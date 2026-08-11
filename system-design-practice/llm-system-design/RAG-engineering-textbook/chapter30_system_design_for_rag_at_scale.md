# Chapter 30: System Design for RAG at Scale

## 30.1 What This Chapter Covers

A RAG pipeline that returns a great answer in a notebook in three seconds is a research result. A RAG pipeline that returns a great answer in under a second, for thousands of concurrent users, without falling over when the vector database has a slow morning — that's a system. This chapter is about the gap between those two things.

We'll look at where end-to-end latency actually goes in a RAG request, how to set a **latency budget** across the stages of the pipeline, where **caching** saves both time and money, and where **async and parallel execution** let you hide work instead of eliminating it. None of this is exotic distributed-systems theory — it's the same discipline any latency-sensitive service needs, applied to a pipeline that happens to have a search engine and a language model bolted together.

---

## 30.2 Where the Time Actually Goes

Recall the four-box mental model from Chapter 1: retrieve, augment, generate. At scale, each of those boxes has real, measurable latency, and they don't cost the same.

A rough breakdown for a typical production RAG request might look like this (illustrative numbers, not a benchmark — your system will differ):

| Stage | What happens | Typical share of latency |
|---|---|---|
| Query understanding (Ch. 13) | Rewriting, intent classification, query expansion | Small — tens of milliseconds |
| Retrieval (Ch. 14) | Embedding the query, searching the vector index, possibly keyword search | Small to moderate |
| Re-ranking (Ch. 15) | Scoring candidates with a cross-encoder or similar | Moderate — can spike if not careful |
| Generation (Ch. 17–18) | The LLM produces the answer, token by token | Large — usually the majority |

The headline fact worth internalizing: **generation dominates.** A cross-encoder re-ranker scoring twenty candidates might take 50–150ms. A vector search over a well-indexed collection might take 10–50ms. But an LLM generating a few hundred tokens can easily take one to several seconds, especially with a large model. If your total budget is, say, 3 seconds, generation alone might consume 2 of them.

This has a direct design consequence: **the earlier stages need to be fast in absolute terms, not just fast relative to generation**, because they're all sequential dependencies before generation can even start. A slow retriever doesn't just cost its own latency — it delays the moment the model can begin producing tokens at all.

---

## 30.3 Setting a Latency Budget

A **latency budget** is simply an agreed-upon time allowance for each stage, derived from your overall target. If product wants "answers start streaming within 2 seconds for 95% of requests," you don't get there by hoping — you allocate.

A worked example:

```
Total budget (p95):           2000 ms
  Query understanding:          50 ms
  Retrieval:                   150 ms
  Re-ranking:                  150 ms
  Prompt assembly:               20 ms
  Time-to-first-token (LLM):   400 ms
  ─────────────────────────────────
  Budget before streaming starts: 770 ms
  Remaining generation happens while streaming to the user
```

Notice the last line: because generation can **stream**, you don't need to budget for the *entire* answer before the user sees anything — only for time-to-first-token. This is one of the most important levers in RAG latency design, and we'll return to it in Section 30.5.

The budget matters because it forces explicit tradeoffs. If re-ranking a large candidate set is pushing you over budget, you now have a concrete number to negotiate against: shrink the candidate pool, use a lighter-weight re-ranker, or cut the stage entirely for latency-sensitive query types. Without a budget, every stage's owner optimizes locally and nobody is accountable for the total.

> **Core idea:** A latency budget turns "make it fast" — an unfalsifiable goal — into a small set of per-stage numbers that can each be measured, monitored, and defended independently.

---

## 30.4 Caching Layers

Caching is the single highest-leverage tool in a RAG system's latency and cost toolbox, because RAG workloads are far more repetitive than they first appear. Popular questions get asked again and again, in slightly different phrasing, by different users.

There are several distinct caching opportunities, and it's worth treating them separately because they have different keys, different invalidation rules, and different failure modes.

**Embedding cache.** Embedding a piece of text is deterministic for a given model version — the same input always produces the same vector. This makes embeddings an easy caching target: cache by a hash of (model version, input text), for both document chunks at ingestion time and, where queries repeat, at query time. This mainly saves compute cost (Chapter 31), but it also removes a network round-trip to an embedding API, which is real latency.

**Retrieval result cache.** If two requests submit the exact same query against the same index version, there's no reason to search twice — cache the ranked chunk IDs (and scores) keyed on (query text, index version, filters). The tricky part is filters: a retrieval cache keyed only on query text will silently return wrong results if the same query is issued with different access-control filters (see Chapter 34) or metadata scopes. The cache key must include everything that can change the result set.

**Semantic caching.** Exact-match caching misses a huge share of real traffic, because users rarely type identical queries. "What's our refund policy?" and "How do I get a refund?" are different strings but the same information need. A **semantic cache** embeds the incoming query and checks whether a sufficiently similar query has been answered recently — if the cosine similarity to a cached query exceeds a threshold, you can reuse the cached retrieval results, or even the cached final answer.

Semantic caching is powerful but carries real risk: a similarity threshold that's too loose will return a cached answer to a question that's subtly different in a way that matters (wrong time period, wrong product tier, wrong customer). Treat the threshold as a tunable, monitored parameter, not a one-time setting — and consider caching only the *retrieval* result rather than the final generated answer, so the LLM still has a chance to notice a mismatch and hedge appropriately.

| Cache type | Caches | Key | Main benefit |
|---|---|---|---|
| Embedding cache | Vector for a text input | hash(model, text) | Compute cost, some latency |
| Retrieval result cache | Ranked chunk IDs/scores | (query, index version, filters) | Latency, retrieval cost |
| Semantic cache | Similar-query results | nearest cached query above threshold | Latency across paraphrases |

---

## 30.5 Async and Parallel Execution

Not every millisecond can be cached away, but plenty of it can be **overlapped** instead of spent sequentially.

**Parallel retrieval strategies.** If your system runs both dense vector search and keyword/BM25 search for hybrid retrieval (Chapter 12), or queries multiple indexes for different content types (Chapter 8), there's no reason to run them one after another. Fire them concurrently and merge results when they all return — the wall-clock cost is the slowest of the parallel branches, not the sum of all of them.

**Parallel re-ranking batches.** Cross-encoder re-ranking scores each (query, candidate) pair independently. Batching these calls and, where the re-ranker supports it, running batches concurrently rather than one candidate at a time, is a straightforward win that's easy to overlook when a pipeline is built up incrementally.

**Streaming generation.** This is the biggest lever of all. Rather than waiting for the LLM to finish the entire answer before showing anything, stream tokens to the user as they're produced. The perceived latency — what the user actually experiences — becomes time-to-first-token, not time-to-last-token. A 3-second total generation time feels dramatically different depending on whether the user sees nothing for 3 seconds or sees the first words appear at 400ms and the rest fill in naturally.

**Overlapping later pipeline stages with generation.** Some work doesn't need to block the user-visible answer at all. Logging the retrieval trace (Chapter 32), computing evaluation signals, or kicking off a background quality check can all happen *after* generation has started streaming, off the critical path entirely.

A useful design habit: for every stage in your pipeline, ask "does the user need to wait for this before seeing anything?" If the answer is no, it doesn't belong in the synchronous, blocking path.

---

## 30.6 A Note on Limits — Latency Engineering Has a Floor

It's tempting to think that with enough caching and parallelism, RAG latency can be driven arbitrarily low. In practice, there's a floor, and it's worth naming honestly:

- **Cold-cache traffic still pays full price.** Caching helps the popular head of the query distribution; the long tail of novel queries always hits the full pipeline. If your traffic is dominated by unique, one-off questions (common in specialized enterprise search), caching's benefit shrinks considerably.
- **Semantic caching trades latency for correctness risk.** Every cache hit on a paraphrased query is a small bet that the paraphrase didn't change the intent. That bet is sometimes wrong, and it fails silently — the user gets a fast, confidently wrong-shaped answer.
- **Generation latency has a hard floor set by model size and output length.** No amount of pipeline engineering upstream changes how long a large model takes to produce five hundred tokens. If your latency target is aggressive, the model choice (and Chapter 31's cost/latency tradeoffs) matters more than anything discussed in this chapter.
- **Parallelism adds operational complexity.** Concurrent retrieval branches, background logging, and speculative work all mean more moving parts to monitor and more ways for a partial failure to produce a partial or inconsistent result.

Good latency engineering buys you a faster typical case and a more honest worst case — it doesn't buy you a system with no worst case.

---

## 30.7 Chapter Summary

- End-to-end RAG latency breaks down across query understanding, retrieval, re-ranking, and generation — and **generation typically dominates**, so it deserves the largest share of the budget and the most scrutiny on model choice.
- A **latency budget** allocates a target time to each pipeline stage, turning a vague "make it fast" goal into concrete, monitorable per-stage numbers.
- **Caching** has several distinct forms — embedding caches, exact-match retrieval result caches, and **semantic caches** for paraphrased queries — each with different keys and different invalidation risks.
- Retrieval and cache keys must include **filters and access-control scope**, not just query text, or caching can silently leak results across contexts.
- **Async and parallel execution** — running retrieval strategies concurrently, batching re-ranking calls, and streaming generation — reduces perceived latency even when it doesn't reduce total work.
- **Streaming** shifts the user-perceived latency target from time-to-last-token to time-to-first-token, which is often the single highest-leverage change available.
- Caching and parallelism reduce the *typical* case but do not remove the *floor* set by cold-cache traffic, model size, and output length — latency engineering has honest limits.

**Coming up next (Chapter 31):** with the latency picture in place, we turn to the other side of running RAG at scale — where the money actually goes, and the practical levers for reducing embedding, retrieval, and token cost without gutting quality.
