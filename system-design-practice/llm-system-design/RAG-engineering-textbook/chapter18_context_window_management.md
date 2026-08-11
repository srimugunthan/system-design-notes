# Chapter 18: Context Window Management

## 18.1 What This Chapter Covers

If retrieval finds ten relevant chunks, why not just hand all ten to the model? Modern context windows can hold tens or even hundreds of thousands of tokens — surely more context can only help.

It turns out this instinct is wrong more often than it's right. This chapter explains why simply stuffing more retrieved text into the prompt is not a free win — the cost, latency, and accuracy tradeoffs involved — and walks through the two practical toolkits for managing this: **compressing** context before it's injected, and **selecting** which context deserves to be there at all.

---

## 18.2 The Temptation of "Just Add More Context"

A large context window feels like it should make retrieval easier: if you're not sure which three chunks are the right ones, why not retrieve twenty and let the model sort it out? The model is good at reading, after all.

This reasoning breaks down for three concrete reasons:

- **Cost.** Most hosted LLMs charge per input token. Retrieving and injecting 20 chunks instead of 5 can multiply the cost of every single query, at scale, for no accuracy benefit if most of those chunks are irrelevant. We'll quantify this tradeoff properly in Chapter 31.
- **Latency.** Larger prompts take longer to process before the model even starts generating a response. In a user-facing chat interface, an extra second or two of "thinking" time before the first token appears is very noticeable.
- **Accuracy — the counterintuitive one.** More context does not reliably mean a better answer. Past a certain point, adding more retrieved chunks can *actively hurt* answer quality, because of an effect worth understanding in detail: **lost in the middle**.

---

## 18.3 The "Lost in the Middle" Problem

Chapter 4 briefly introduced the idea that LLMs don't treat every position in their context window equally. This chapter is where we unpack it properly.

Empirically, LLMs tend to make best use of information placed at the **very beginning** or the **very end** of their context window, and are measurably worse at recalling or using information buried in the **middle** of a long prompt — even when that middle information is just as relevant as what's at the edges. This is often visualized as a U-shaped curve: performance is high near both ends of the context and dips in the middle.

Think of it like reading a long legal contract under time pressure. You read the opening clauses carefully. You read the closing clauses carefully, because that's where the enforceable terms usually land. The forty pages of dense boilerplate in between get skimmed, and if the one clause that actually matters happens to be buried on page 22, there's a real chance you miss it — not because you couldn't understand it, but because of *where it was sitting*.

This has a direct, practical consequence for RAG: **the order in which you inject retrieved chunks into the prompt is not a cosmetic detail — it materially affects whether the model actually uses your best evidence.** A retriever that correctly finds the single most relevant chunk, and then a prompt template that buries it as chunk #6 out of 10, can produce a worse answer than a system with mediocre retrieval but smart placement.

> **Key idea:** context window size tells you how much information a model *can* hold. It says nothing about how evenly the model actually *attends to* that information. Those are two different properties, and RAG engineering has to account for both.

---

## 18.4 Selective Inclusion: Deciding What Earns a Seat in the Prompt

If more context isn't automatically better, the natural next question is: which retrieved chunks actually deserve to be in the prompt? A few strategies, usually combined rather than used in isolation:

**Relevance thresholds.** Every retrieved chunk typically comes with a similarity or relevance score (Chapter 14) or a re-ranker score (Chapter 15). Instead of always injecting a fixed top-k, drop any chunk below a minimum score threshold — a weak 6th match is often worse than no 6th match at all, since it adds noise and token cost without adding useful evidence.

**Deduplication.** It's common for a knowledge base to contain multiple chunks that say nearly the same thing — an FAQ entry and a paragraph from a policy document both describing the same refund rule, for instance. Injecting both wastes context budget and, in the worst case, makes the model think it has more independent confirmation of a fact than it actually does. Near-duplicate detection (via embedding similarity between candidate chunks, or simpler text-overlap heuristics) before injection keeps the context lean.

**Diversity-aware selection.** The flip side of deduplication: if the top 5 retrieved chunks are all near-duplicates of each other, the system may have completely missed a differently-worded but highly relevant chunk sitting at rank 8. Techniques like Maximal Marginal Relevance (MMR) explicitly balance relevance against diversity when selecting the final chunk set, rather than greedily taking the top-k by score alone.

**Query-adaptive chunk count.** Not every question needs the same amount of context. A narrow factual question ("what's the refund window for international orders?") may be fully answerable from one chunk. A broad synthesis question ("summarize our current refund policies across regions") genuinely needs more. Some systems vary k dynamically based on query characteristics rather than using a fixed k for every request.

| Strategy | What it removes/reorders | Primary benefit |
|---|---|---|
| Relevance threshold | Low-scoring chunks | Cuts noise and token cost |
| Deduplication | Near-identical chunks | Frees budget for genuinely new information |
| Diversity-aware selection (MMR) | Redundant top-ranked chunks | Surfaces differently-worded relevant content |
| Reordering (edges vs. middle) | Nothing removed, order changed | Mitigates lost-in-the-middle |
| Query-adaptive k | Chunk count itself | Matches context size to question complexity |

---

## 18.5 Reordering: Fighting Back Against the Middle

Given the lost-in-the-middle effect from 18.3, one of the cheapest, highest-leverage fixes available is simply **reordering** the chunks you've already decided to include — no retrieval or model changes required.

A common pattern: place the highest-scoring chunk either **first** or **last** in the context block (some teams put the single best chunk last, right before the question, on the theory that "most recently read" carries extra weight for the immediate answer), and push the lower-confidence supporting chunks toward the middle where their loss is least costly. This is sometimes called a "**best-worst-best**" ordering — strongest evidence at both edges, weaker evidence sandwiched in between.

This is a genuinely counterintuitive engineering lever: two systems can retrieve the *exact same set* of chunks and produce meaningfully different answer quality purely because of injection order. It's also one of the cheapest experiments a team can run — reordering requires no changes to the retriever or the index, only to the prompt assembly step, which makes it a good first thing to try when an otherwise well-retrieved answer is missing information that was clearly present in the context.

---

## 18.6 Context Compression

Selective inclusion decides *which* chunks to keep. Compression asks a different question: can we keep the *information* from a chunk while spending fewer tokens on it?

A few conceptual approaches, roughly in increasing order of aggressiveness:

- **Trimming.** Many retrieved chunks contain boilerplate — headers, navigation text, repeated disclaimers — that survived document parsing (Chapter 5) but carries no answer-relevant information. Stripping this before injection is close to a free win.
- **Extractive compression.** Instead of injecting a full chunk, extract only the sentences within it that are actually relevant to the query, using a lightweight relevance-scoring pass over sentences. This keeps the original wording (preserving traceability for citations) while cutting token count substantially.
- **Abstractive summarization.** Use a smaller, cheaper model to summarize a chunk (or a cluster of chunks) before it's handed to the main generator. This can dramatically shrink token usage, but it introduces a real risk: the summarization step is itself a generation step, and it can drop nuance, misstate a number, or introduce its own small hallucination *before* the main model ever sees the source text. Compression that corrupts the evidence is worse than no compression at all.
- **Query-focused compression.** A refinement of the above: rather than summarizing a chunk in general, summarize it specifically with respect to the current question, discarding content in the chunk that's unrelated to what's being asked.

The tradeoff running through all of these is the same one that shows up throughout this book: **every compression step trades fidelity for efficiency**, and the right amount of compression depends on how much fidelity your use case can afford to lose. A system answering casual product questions can tolerate lossier compression than one generating summaries used in regulated financial disclosures (Chapter 35).

---

## 18.7 What Context Management Doesn't Fix

It's worth being direct about the limits here, because it's easy to over-invest in context engineering as if it were a substitute for good retrieval and indexing:

- **No amount of reordering fixes context that shouldn't have been retrieved in the first place.** Compression and reordering operate on whatever the retriever handed them; garbage in, cleverly-arranged garbage out.
- **Compression can introduce its own errors.** A summarization step that misrepresents a source is arguably worse than an unsummarized chunk that's merely long, because it corrupts the evidence the model is meant to ground itself in.
- **Lost-in-the-middle mitigation is a mitigation, not a cure.** Even with best-worst-best ordering, very long contexts still generally underperform well-curated shorter ones. Bigger context windows expand what's *possible*, not what's *optimal*.
- **These techniques add engineering surface area.** Deduplication thresholds, MMR parameters, and compression models all need tuning and monitoring; they are additional moving parts that can silently degrade over time if left unmonitored (a topic for Chapter 32).

---

## 18.8 Chapter Summary

- More retrieved context is **not automatically better** — it increases cost and latency, and past a point can reduce answer quality.
- The **lost in the middle** effect means models attend more reliably to information at the start and end of their context window than information buried in the middle, regardless of that information's relevance.
- **Selective inclusion** — relevance thresholds, deduplication, diversity-aware selection (MMR), and query-adaptive chunk counts — decides which retrieved chunks earn a place in the prompt at all.
- **Reordering** retrieved chunks (best-worst-best) is a low-cost, high-leverage fix for lost-in-the-middle that requires no changes to retrieval itself.
- **Context compression** — trimming, extractive compression, abstractive summarization, and query-focused summarization — reduces token cost but trades away some fidelity, and summarization in particular can introduce new errors before the main model ever generates an answer.
- These techniques manage context that was already retrieved; they **cannot substitute for retrieval and indexing quality** upstream.

**Coming up next (Chapter 19):** we'll tackle what happens when the context you've carefully selected and arranged actually disagrees with itself — outdated versus current documents, conflicting policies, and how to keep a RAG system from silently picking a side.
