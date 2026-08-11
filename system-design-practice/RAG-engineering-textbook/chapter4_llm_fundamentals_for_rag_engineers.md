# Chapter 4: LLM Fundamentals for RAG Engineers

## 4.1 What This Chapter Covers

You don't need a PhD in deep learning to build a good RAG system, but you do need a working, practical understanding of a few things about how LLMs actually process text — because these details directly shape decisions you'll make throughout this book: how big to make your chunks, how many chunks you can afford to retrieve, and where to place them in the prompt.

This chapter is deliberately narrow. We're not covering how transformers are trained or the mathematics of self-attention. We're covering exactly what a RAG engineer needs to reason correctly about **tokens, context windows, prompt structure, and why position in a prompt matters** — the concepts that Chapters 2 and 3 have already been leaning on without fully explaining.

---

## 4.2 Tokens: The Real Unit of Text an LLM Sees

When people talk about the length of a document, they usually think in words or characters. LLMs don't. They operate on **tokens** — chunks of text, typically a few characters long, that the model's tokenizer breaks language into before processing it.

A rough mental model:

- Common English words are often a single token ("the", "cat", "running")
- Longer or less common words often split into multiple tokens ("retrieval" might become "retriev" + "al")
- Punctuation, whitespace, and even parts of numbers can each be their own token
- As a loose rule of thumb for English text, **1 token is roughly 3/4 of a word**, or about 4 characters — but this varies by language, tokenizer, and content type (code and non-English text often tokenize less efficiently)

> **Core idea:** Every number an LLM provider gives you — context window size, price per query, rate limits — is measured in tokens, not words or characters. If you're estimating cost or capacity using word counts, your numbers will be off.

Why this matters for RAG specifically: when you chunk a document (Chapter 6), decide how many chunks to retrieve (Chapter 14), or estimate what a query will cost, you're really doing arithmetic in tokens, not words. A "500-word chunk" and a "500-token chunk" are meaningfully different sizes, and mixing the two up is a common source of miscalculated costs and unexpectedly truncated prompts.

---

## 4.3 Why Token Count Governs Cost and Limits

Two of the most practical numbers in your entire RAG system trace directly back to token counts:

**Cost.** Most LLM APIs price usage per token — typically with a separate rate for tokens you send in (the prompt, including your retrieved chunks) and tokens the model generates in response. A RAG system that retrieves ten large chunks per query, on every query, at high volume, is not a rounding error — it's a direct, measurable cost that scales with retrieval choices you control. This is why Chapter 2 flagged that stuffing huge amounts of raw text into a prompt (the long-context approach) can get expensive fast, and it's the starting point for the deeper cost discussion in Chapter 31.

**Limits.** Every model has a maximum number of tokens it can process in a single request — its **context window**. Send more tokens than that limit, and the request either fails outright or gets silently truncated, depending on the API. Either outcome is bad: a failed request is at least visible, but a silently truncated prompt might quietly drop your most important retrieved chunk without any error at all.

---

## 4.4 The Context Window: Your Retrieval Budget

The **context window** is the total number of tokens a model can consider at once — system instructions, conversation history, retrieved chunks, and the model's own response, all sharing the same fixed budget.

For a RAG engineer, this is worth reframing in blunt, practical terms: **the context window is your retrieval budget.** Every token spent on system instructions or conversation history is a token not available for retrieved content, and every token spent on retrieved content is a token not available for the model's answer.

A simplified example of how that budget gets divided, illustrative rather than exact:

| Component | Illustrative share of a modest context window |
|---|---|
| System instructions | Small, fixed cost |
| Conversation history (if any) | Grows with a multi-turn chat |
| Retrieved chunks | The main variable — this is what retrieval and augmentation control |
| Reserved space for the model's answer | Must be reserved *before* the call, not after |

This budget is exactly why Chapter 3 emphasized that chunking decisions matter so much: if your chunks are large, you can fit fewer of them before the context window fills up. If they're small, you can fit more — but each one carries less surrounding context on its own. Chapter 18 covers strategies for managing this budget under real constraints, including what to do when relevant content simply doesn't fit.

---

## 4.5 Prompt Structure in a RAG System

Modern LLM APIs typically accept prompts as a small set of distinct sections rather than one undifferentiated blob of text. A typical RAG prompt is assembled from roughly three parts:

- **System section** — standing instructions that apply to every query: the model's role, tone, and rules (e.g., "You are a support assistant. Only answer using the provided context. If the context doesn't contain the answer, say so.")
- **Context section** — the retrieved chunks from Stage 3 of the pipeline (Chapter 3), usually labeled with metadata like source or date so the model (and, ideally, the end user) can tell where each piece of information came from
- **User section** — the actual question being asked, and often recent conversation history if the system is a multi-turn chat

Roughly:

```
[SYSTEM]
You are an assistant that answers questions using only the
provided context. If the answer isn't in the context, say so.

[CONTEXT]
[Source: refund-policy-2024.pdf, Section 3]
Refunds must be requested within 30 days of purchase...

[Source: refund-policy-2024.pdf, Section 5]
Digital goods are non-refundable except in cases of...

[USER]
Can I get a refund on a digital purchase after 45 days?
```

This structure is what Chapter 3 called "augmentation" — and getting it right is less about clever wording and more about discipline: keeping instructions separate from retrieved facts, labeling sources consistently, and being explicit about what the model should do when the context doesn't contain an answer. Chapter 17 goes deep on prompt engineering specifically for RAG.

---

## 4.6 A Plain-Language Intuition for Attention

You don't need the math to build good intuition here, so let's build it with an analogy instead of equations.

Imagine you're handed a stack of ten documents and asked a question. You don't read every word with equal focus — you skim, and your attention naturally jumps to the parts that seem most relevant to the question, wherever they happen to sit in the stack. That's roughly what happens inside a transformer-based LLM: for every word it's about to generate, the model computes how much "attention" to pay to every other piece of text in its context, and weighs its answer accordingly. This mechanism — called **self-attention** — is what lets the model connect a word at the end of a long prompt to a fact mentioned near the beginning.

In principle, this means position in the prompt shouldn't matter — the model can, in theory, attend to anything, anywhere in the context. In practice, it isn't that clean.

---

## 4.7 Why Position in the Context Still Matters

Empirically, models tend to be more reliable at using information that sits near the **beginning** or the **end** of a long context, and less reliable at using information buried in the **middle** — even when that information is fully present in the prompt and technically available for the model to attend to. This pattern is informally known as **"lost in the middle,"** and it has been observed across many model families and context lengths.

Think of it like reading a long meeting transcript: you tend to remember the opening framing and the closing conclusions clearly, while a key detail mentioned in the middle of a long, meandering discussion is easier to lose track of — even though you technically "heard" it. LLMs show a version of this same pattern.

Why does this matter for RAG specifically?

- If your augmentation step (Chapter 3, Chapter 17) dumps retrieved chunks into the prompt in an arbitrary order, your *most relevant* chunk might land in the worst possible position for the model to use it well
- Simply retrieving the right information is not the same as the model actually *using* that information correctly — retrieval quality and generation quality are related but separate concerns
- This is one of several reasons "just widen the context window and retrieve more" (the long-context approach from Chapter 2) doesn't automatically translate into better answers

We're only building the intuition here — the mechanics, evidence, and concrete mitigations (like deliberately placing the most important chunk near the start or end of the prompt) belong to Chapter 18.

---

## 4.8 A Note of Honesty: This Is a Working Model, Not the Full Picture

Everything in this chapter is simplified on purpose, and it's worth being upfront about what's been left out:

- Real tokenizers behave inconsistently across languages, code, and special characters in ways this chapter didn't cover
- "Lost in the middle" is an empirical pattern observed in research and practice, not a fixed law — its severity varies by model, and newer models are actively being trained to mitigate it, so treat it as a tendency to design around rather than an absolute rule
- Context window size, token pricing, and attention behavior all differ meaningfully between model providers and even between versions of the same model family — the qualitative ideas here hold up, but the specific numbers you'll work with will come from whichever model you're actually using
- None of this is a substitute for reading your chosen model provider's own documentation on tokenization and context limits when you're making precise cost or capacity calculations

What this chapter *does* give you is enough shared vocabulary — tokens, context window, prompt structure, and the lost-in-the-middle pattern — to read the rest of this book without those terms feeling like unexplained jargon.

---

## 4.9 Chapter Summary

- LLMs process text as **tokens**, not words or characters — token count, not word count, governs both cost and context limits.
- The **context window** is a fixed token budget shared across system instructions, conversation history, retrieved chunks, and the model's response — it functions as your retrieval budget.
- A typical RAG prompt is assembled from **system, context, and user sections**, with retrieved chunks ideally labeled by source so answers can be traced back to their origin.
- **Self-attention** lets a model in principle relate any part of its context to any other part, which is why long-context prompts can work at all.
- In practice, models tend to use information near the **beginning and end** of a long context more reliably than information buried in the **middle** — the "lost in the middle" pattern, which later chapters address directly.
- These fundamentals directly inform practical RAG decisions: chunk sizing, how many chunks to retrieve, how to order them in a prompt, and when a long-context approach is worth its cost.

**Coming up next (Chapter 5):** with the foundations of Part I in place — why RAG exists, how it compares to fine-tuning and long-context, the shape of a full pipeline, and how LLMs consume text — we move into Part II and start at the very first stage of that pipeline: parsing raw, messy real-world documents into clean text a RAG system can actually use.
