# Chapter 15: Re-ranking

## 15.1 What This Chapter Covers

Retrieval, as built in Chapter 14, gets you a shortlist of plausible chunks fast — often across millions of candidates, in milliseconds. But "fast and plausible" is not the same as "accurate." This chapter asks: once we have a shortlist, how do we get more precise about which chunks in it actually deserve to be handed to the generator?

The answer is **re-ranking** — a second, slower pass over a small candidate set, using a more accurate (and more expensive) method than the one that produced the shortlist in the first place. We'll cover cross-encoders, LLM-based re-ranking, and the general "cascade" architecture that ties fast and slow retrieval together.

---

## 15.2 Why the First Pass Is Fast but Imprecise

To understand why re-ranking exists, we need to be precise about what the first-pass retriever (Chapters 9–14) is actually doing.

Standard vector search uses what's called a **bi-encoder**: the query and each document chunk are each embedded *separately*, into fixed-length vectors, completely independent of one another. The document embeddings are computed once, ahead of time, and stored in the index (Chapter 10). At query time, the query is embedded, and similarity (usually cosine similarity) is computed between the query vector and every candidate document vector.

This separateness is exactly what makes bi-encoders fast — document embeddings are precomputed, so retrieval at query time is just a nearest-neighbor search (Chapter 11) against vectors that already exist. You can search millions or billions of chunks in milliseconds this way.

But that same separateness is also the source of its imprecision. Because the query and the document are never looked at *together*, the model never gets to reason about how specific words in the query interact with specific words in the document. It only compares two independently-computed summaries of "what this text is about." Two chunks can end up with very similar embeddings despite being relevant for different reasons, or a chunk can score moderately well on overall topical similarity while missing the one specific detail the query actually needed.

> A bi-encoder answers the question "are these two pieces of text generally about the same thing?" It is much weaker at answering the sharper question "does this specific document actually answer this specific question?" — and that sharper question is the one that matters for generation quality.

This isn't a flaw to be fixed — it's an inherent tradeoff of the architecture. Bi-encoders are fast *because* they don't look at the query and document jointly. Getting joint reasoning back requires a different kind of model, applied to a much smaller set of candidates.

---

## 15.3 Cross-Encoders: Joint Reasoning Over Query and Document

A **cross-encoder** is a model that takes the query and a single candidate document *together*, as one combined input, and outputs a relevance score for that pair directly — rather than comparing two independently-computed vectors.

```
Bi-encoder (first pass):
  Query  ──► embed ──► vector A  ─┐
                                   ├── compare (cosine similarity)
  Doc    ──► embed ──► vector B  ─┘

Cross-encoder (re-ranking pass):
  [Query + Document] ──► single model ──► relevance score
  (query and document are read together, in the same forward pass)
```

Because the cross-encoder processes the query and document jointly, it can pick up on fine-grained interactions — whether a specific term in the query is actually addressed in the document, whether the document's answer matches the query's implied scope, whether negation or qualifiers change the meaning. This tends to make cross-encoders meaningfully more accurate at judging true relevance than bi-encoder similarity alone.

The catch is cost. A cross-encoder can't precompute document representations ahead of time, because its input depends on the specific query paired with each document — the whole point is that query and document are read together. That means scoring N documents requires N full forward passes through the model *at query time*, every single time. Running a cross-encoder over your entire index the way you'd run a bi-encoder search is computationally impractical.

This is why cross-encoders are never used as the first retrieval pass. They're used as a **second pass**, applied only to the small shortlist (typically tens of candidates) that the fast bi-encoder search already narrowed things down to. Scoring 20–50 candidates with a cross-encoder is entirely tractable; scoring millions is not.

---

## 15.4 LLM-Based Re-ranking

A cross-encoder is a purpose-built, relatively small model trained specifically to score query-document relevance. **LLM-based re-ranking** uses a general-purpose large language model for the same job instead — typically by prompting it with the query and a candidate chunk (or a batch of candidates) and asking it to score, rate, or directly reorder them by relevance.

This can look as simple as:

```
Prompt: Given the question and the passage below, rate how well the
passage answers the question on a scale of 1-10.

Question: {query}
Passage: {candidate chunk}
```

...repeated (or batched) across the shortlist, with results sorted by score.

LLM-based re-ranking has a few notable advantages over cross-encoders:

- **No dedicated re-ranking model to train or host** — you're reusing an LLM you likely already have access to
- **Better reasoning about nuance** — a capable LLM can weigh context, qualifiers, and implicit intent in ways a smaller purpose-built cross-encoder may miss
- **Flexible criteria** — the prompt can be adjusted to rerank by recency, specificity, authority, or any other criterion you can describe in words, not just generic relevance

But the tradeoffs are significant, and worth naming plainly:

- **Latency** — LLM calls are typically much slower than a small cross-encoder's forward pass, especially at any meaningful batch size
- **Cost** — scoring even 20–30 candidates per query with a hosted LLM adds up fast at production query volumes, a concern we'll quantify more carefully in Chapter 31
- **Consistency** — LLM-assigned scores can be less stable and less calibrated than a model trained explicitly and only for relevance scoring

In practice, LLM-based re-ranking tends to show up either in lower-volume, higher-stakes applications where accuracy is worth the cost (e.g., some financial services workflows discussed in Chapter 35), or as a final, very-small-shortlist pass (reranking the top 5, not the top 50) layered after a cheaper cross-encoder has already done the bulk of the narrowing.

---

## 15.5 The Cascade Pattern

Both cross-encoders and LLM-based re-ranking share the same underlying architectural idea, which shows up constantly in retrieval systems once you know to look for it: **do the cheap, approximate thing first over a large set, then do the expensive, accurate thing second over a small set.** This is often called a **cascade** or **funnel** architecture.

```
Index: millions of chunks
      │
      ▼
[Stage 1] Bi-encoder vector search (fast, approximate)
      │  narrows millions → hundreds
      ▼
[Stage 2] Cross-encoder re-ranking (slower, more accurate)
      │  narrows hundreds → tens
      ▼
[Stage 3] LLM-based re-ranking (slowest, most accurate) — optional
      │  narrows tens → the final few (k)
      ▼
Chunks sent to the generator
```

Each stage in the cascade is more accurate but more computationally expensive than the one before it, and each stage operates on a smaller candidate set than the one before it — which is exactly what makes the increasing expense affordable. You'd never run a cross-encoder over an entire index, but running it over the 200 candidates a vector search already surfaced is entirely reasonable. The same logic applies again if you add an LLM-based re-ranking stage on top: expensive, but only over the handful of candidates the cross-encoder already vouched for.

Not every system needs all three stages. A latency-sensitive application might stop after Stage 1 (accepting lower precision for speed), or after Stage 2 (a very common production setup — bi-encoder retrieval plus a cross-encoder re-rank is often the highest-leverage two-stage combination). A high-stakes, lower-volume application might run the full three-stage cascade. The right number of stages is an engineering decision, not a fixed rule, made against the latency and cost budgets we'll return to in Part VIII.

---

## 15.6 A Note of Honesty: Re-ranking Cannot Rescue a Bad Retrieval Set

It's worth being blunt about the ceiling here: **re-ranking can only reorder what the first pass already found.** If the correct document never made it into the initial shortlist — because the bi-encoder's embedding missed it, or k was set too small, or the query itself was poorly formed (Chapter 13) — no amount of re-ranking sophistication will surface it. Re-ranking improves *precision within the shortlist*; it does nothing for *recall* if the right answer was never a candidate in the first place.

A few other honest caveats:

- **Re-ranking adds latency to every single query**, not just the ones that need it. That cost is paid unconditionally, even on easy queries the first pass already nailed.
- **Cross-encoders need training data too**, and an off-the-shelf cross-encoder trained on generic web relevance may not transfer perfectly to a narrow technical or regulatory domain without some fine-tuning or careful evaluation.
- **LLM-based re-ranking inherits LLM quirks** — including a tendency to be swayed by surface-level fluency or length rather than genuine relevance, if the prompt isn't carefully designed and tested.

The practical takeaway: invest in a wide, high-recall first pass (a large enough k, good chunking, good embeddings) before leaning on re-ranking to clean things up. Re-ranking is a precision tool, not a recall fix.

---

## 15.7 Chapter Summary

- The fast first-pass retriever (a **bi-encoder**) embeds queries and documents separately, which is what makes it fast — and also what limits its precision, since query and document are never reasoned about jointly.
- A **cross-encoder** reads the query and a candidate document together in a single forward pass, producing a more accurate relevance score, but at a cost that makes it impractical to run over an entire index — only over a small shortlist.
- **LLM-based re-ranking** uses a general-purpose LLM to score or reorder candidates, offering flexible criteria and strong reasoning at the cost of higher latency, higher cost, and potentially less consistent scoring than a dedicated cross-encoder.
- Production systems typically use a **cascade** (or funnel) architecture: cheap, approximate retrieval over a large candidate set, followed by one or more increasingly expensive, increasingly accurate re-ranking stages over progressively smaller sets.
- Re-ranking improves precision on the candidates already retrieved — it cannot recover relevant documents that the first pass failed to surface in the first place.

**Coming up next (Chapter 16):** re-ranking assumes the answer exists somewhere in a single retrieved set. Some questions can't be answered that way at all — they require chaining facts across multiple documents in sequence. Chapter 16 covers multi-hop and iterative retrieval for exactly these cases.
