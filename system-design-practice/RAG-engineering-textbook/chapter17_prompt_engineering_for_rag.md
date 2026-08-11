# Chapter 17: Prompt Engineering for RAG

## 17.1 What This Chapter Covers

By this point in the book, we've built a pipeline that can find the right pieces of text (Part IV). But finding the right information and getting the model to *use it correctly* are two different problems. Hand a language model a pile of retrieved text and a question with no further guidance, and it will often do something unhelpful — blend the retrieved facts with its own memorized knowledge, ignore half the context, or answer confidently even when the context doesn't actually support an answer.

This chapter is about the layer that sits between retrieval and generation: **the prompt**. We'll look at how a RAG prompt is structured differently from an ordinary chat prompt, how to format retrieved context so the model can actually parse it, how to get per-claim citations, and how to write grounding instructions that meaningfully reduce hallucination rather than just papering over it.

---

## 17.2 A RAG Prompt Is Not a Chat Prompt

A plain chat prompt is simple: a system message describing the assistant's persona, and a user message with a question. The model answers from parametric memory (Chapter 1).

A RAG prompt has an extra, critical ingredient: a block of retrieved evidence that didn't exist when the model was trained, and that changes on every single request. This creates a structural requirement that plain chat prompts don't have — the prompt must clearly separate three distinct things:

1. **Instructions** — how the model should behave (tone, format, grounding rules)
2. **Retrieved context** — the evidence it should reason over
3. **The user's question** — what it's actually being asked

> If the model can't tell where the instructions end and the "evidence" begins, it will treat everything as equally authoritative — including any text that happens to be sitting inside a retrieved document, which is exactly how prompt injection attacks work (more on this in Chapter 34).

A useful mental model: think of the prompt as a **legal brief handed to a judge**. The judge (the model) needs to clearly see which parts are instructions from the court clerk ("weigh only the submitted evidence"), which parts are exhibits (the retrieved chunks), and which part is the actual question being decided. Mix them together into one undifferentiated wall of text, and the judge starts guessing at boundaries — exactly what we don't want.

---

## 17.3 Anatomy of a Well-Structured RAG Prompt

Most production RAG prompts converge on a similar shape, regardless of the underlying model:

```
[SYSTEM / INSTRUCTIONS]
You are a support assistant. Answer using ONLY the provided context.
If the context does not contain the answer, say so explicitly.
Cite the source number for every factual claim.

[CONTEXT]
<retrieved chunks go here, clearly delimited>

[USER QUESTION]
<the actual question>
```

The instructions block generally covers:
- The assistant's role and tone
- The grounding rule (use only the provided context — see 17.5)
- The citation format expected (see 17.4)
- What to do when context is insufficient
- Any output format constraints (we'll go much deeper on this in Chapter 20)

Keeping these as a distinct, front-loaded block matters more than it might seem. Instructions placed *before* the context tend to be followed more reliably than instructions appended after a long block of retrieved text — the model has already spent its "attention budget" processing the context by the time it reaches trailing instructions. We'll return to this attention behavior in more depth in Chapter 18.

---

## 17.4 Context Injection Patterns

How you format the retrieved chunks inside the context block has a real, measurable effect on how well the model uses them. A wall of undifferentiated paragraphs, all run together, makes it hard for the model to tell where one source ends and another begins — which in turn makes accurate citation nearly impossible. Two patterns dominate in practice.

**Numbered source blocks.** Each retrieved chunk is labeled with an index and, ideally, its origin:

```
[Source 1] (refund_policy.md, updated 2026-03-01)
Refunds are issued within 5 business days of the return
being received at our warehouse.

[Source 2] (shipping_faq.md, updated 2025-11-14)
International orders may take up to 14 business days
to process a refund due to customs clearance.
```

**XML-like tags around each chunk.** Functionally similar, but often easier for the model to parse reliably, especially for models trained with a lot of structured or code-adjacent data:

```xml
<source id="1" title="refund_policy.md" date="2026-03-01">
Refunds are issued within 5 business days of the return
being received at our warehouse.
</source>
<source id="2" title="shipping_faq.md" date="2025-11-14">
International orders may take up to 14 business days
to process a refund due to customs clearance.
</source>
```

Both patterns share the same underlying goal: give every chunk a **stable, citable identifier** and attach whatever metadata (source name, date, author, section) the model might need to reason about *which* source to trust — a theme we'll pick back up in Chapter 19 when sources disagree.

| Pattern | Pros | Cons |
|---|---|---|
| Numbered plain-text blocks | Simple, token-efficient, easy to read in logs | Boundaries can blur with long chunks |
| XML-like tags | Very clear boundaries, easy to parse programmatically | Slightly more tokens per chunk |
| JSON array of chunks | Machine-clean, pairs well with structured output (Chapter 20) | Least "natural" for the model to read as prose |

There's no single universally correct choice — teams generally pick one, standardize it across the whole system, and stick with it so the model (and the engineers debugging it) always know what to expect.

---

## 17.5 Grounding Instructions: Telling the Model to Stay on the Page

Grounding is the instruction layer that tells the model *how* to relate to the context it's been given. Weak or missing grounding instructions are one of the most common reasons a technically correct retrieval step still produces a hallucinated answer — the retriever did its job, and the model ignored it anyway.

A few grounding instructions that consistently make a measurable difference in practice:

- **Explicitly permit "I don't know."** Models are trained to be helpful, which biases them toward attempting an answer even from thin evidence. Explicitly telling the model that saying "the provided context doesn't contain this information" is an acceptable, even preferred, answer removes that pressure.
- **Forbid outside knowledge for factual claims.** Something like: *"Answer using only the information in the provided sources. Do not use any knowledge you may already have about this topic."* This directly targets the blending-of-memorized-and-retrieved-facts failure mode described in Chapter 1.
- **Ask for source attribution before the final answer.** Some teams find that asking the model to first list which sources are relevant, and only then compose an answer, produces more grounded output — a lightweight form of forcing the model to "show its work."
- **Set an explicit confidence bar.** For example: *"If only partial information is available, answer only the part you can support and say what's missing."* This avoids the all-or-nothing failure where the model either fabricates the missing piece or refuses to answer anything at all.

> **Core idea:** grounding instructions don't make the model smarter — they change what the model is *rewarded for producing* in this specific interaction: cautious, source-backed text instead of fluent, confident guessing.

A worked example of a compact but effective grounding instruction block:

```
Answer the user's question using ONLY the sources provided below.
- If the sources do not contain enough information to answer,
  respond: "I don't have enough information in the provided
  sources to answer this."
- Do not use prior knowledge to fill gaps.
- Every factual sentence must end with a citation like [1] or [2].
- If sources conflict, say so explicitly rather than picking one.
```

---

## 17.6 Citation Formatting: Making Claims Traceable

Citations are what let a RAG answer be *checked* rather than just trusted — arguably the single biggest practical advantage RAG has over a plain LLM answer. But citation quality depends entirely on how clearly you ask for it.

Common citation formats, roughly in order of how widely they're used:

- **Inline bracket numbers**, e.g., "Refunds take 5 business days [1]." — maps directly to the numbered source blocks from 17.4.
- **Footnote-style**, where citations are collected at the end of the answer rather than inline — easier to read, harder to verify claim-by-claim.
- **Structured citation objects**, where the model returns a JSON field pairing each claim with a source ID — the approach we'll build on heavily in Chapter 20 for machine-consumable output.

The instruction that tends to work best is specific and mechanical rather than vague: not "cite your sources" (too easy to satisfy with one citation at the very end) but "**every sentence containing a factual claim must end with a bracketed citation referencing the source number it came from**." Specificity here is doing real work — a vague citation instruction produces a decorative citation, tacked onto the end of a paragraph as an afterthought rather than tied to individual claims.

It's worth being honest that citation instructions are not self-enforcing: a model can still cite a source that doesn't actually support the sentence it's attached to. Verifying that citations are *correct*, not merely *present*, is an evaluation problem, and we'll cover techniques for measuring citation faithfulness in Chapter 28.

---

## 17.7 Prompt Engineering Is Not a Substitute for Good Retrieval

It's tempting, once you see how much a well-written prompt improves output quality, to treat prompt engineering as a cure-all. It isn't. A few honest limits worth internalizing:

- **No prompt can ground an answer in context that was never retrieved.** If the right document never made it into the top-k results, the most carefully worded grounding instruction just produces a well-formatted "I don't know" — which is *better* than a hallucination, but it's not a fix for a broken retriever.
- **Instructions compete with context for attention**, especially as context grows long (the subject of Chapter 18). A perfect instruction block can still be partially ignored if it's buried under thousands of tokens of retrieved text.
- **Models don't always follow formatting instructions perfectly.** Citation numbers get dropped, "I don't know" gets skipped in favor of a soft guess. Prompt engineering shifts the odds meaningfully in your favor; it does not guarantee compliance, which is why production systems pair it with automated evaluation (Part VII) rather than trusting the prompt alone.
- **Over-constraining the prompt can hurt fluency.** Piling on rules ("cite every sentence, never use outside knowledge, always hedge, always list sources first...") can produce stilted, overly cautious answers. Good grounding instructions are precise, not exhaustive.

---

## 17.8 Chapter Summary

- A RAG prompt must clearly separate **instructions**, **retrieved context**, and the **user question** — blurring these boundaries invites the model to treat everything as equally authoritative.
- **Context injection patterns** like numbered source blocks or XML-like tags give each retrieved chunk a stable, citable identity and make source boundaries unambiguous.
- **Grounding instructions** — explicitly permitting "I don't know," forbidding outside knowledge, requiring citations per claim — directly target the hallucination and blending failure modes introduced in Chapter 1.
- Effective citation instructions are **specific and mechanical** (per-sentence citation requirements) rather than vague ("cite your sources"), which tends to produce decorative rather than useful citations.
- Citations being *present* is not the same as citations being *correct* — verifying faithfulness is an evaluation task, covered in Chapter 28.
- Prompt engineering **cannot compensate for a retriever that missed the right document**, and its instructions compete for the model's attention against the size of the retrieved context.

**Coming up next (Chapter 18):** we'll dig into what happens as the retrieved context grows — why stuffing in more chunks isn't a free win, the "lost in the middle" effect, and practical strategies for compressing and selecting context before it ever reaches the prompt.
