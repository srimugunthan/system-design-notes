# Chapter 14: Retrieval Strategies

## 14.1 What This Chapter Covers

Say you've built a solid index (Part III) and cleaned up the query (Chapter 13). Now comes the part most people picture when they hear "RAG": actually pulling chunks out of the index. But there isn't just one way to do this — and the naive default, taken uncritically, has a failure mode that quietly hurts a lot of production systems.

This chapter walks through four retrieval strategies — **top-k retrieval**, **MMR**, **multi-query retrieval**, and **parent-document retrieval** — and the specific problem each one exists to solve.

---

## 14.2 The Baseline: Top-k Retrieval

The default retrieval strategy in almost every RAG tutorial is **top-k retrieval**: embed the query, compute similarity against every chunk in the index (or an approximation of it, via the indexing algorithms in Chapter 11), and return the *k* chunks with the highest similarity score. If k is 5, you get the 5 closest chunks. Simple, fast, and a completely reasonable place to start.

```
Query embedding
      │
      ▼
Compare against all chunk embeddings (via ANN index)
      │
      ▼
Sort by similarity score, descending
      │
      ▼
Return top k chunks
```

The problem is what "closest" tends to mean in practice. Similarity search finds chunks that are *similar to the query* — but it says nothing about whether those chunks are similar *to each other*. And very often, they are.

**The redundancy problem:** imagine a knowledge base with a policy document that repeats its key point in the introduction, the body, and the FAQ section at the end — worded slightly differently each time, but conceptually identical. All three passages are highly similar to a relevant query, so top-k retrieval with k=5 might return three or four near-duplicate chunks that all say essentially the same thing, crowding out other chunks that would have added genuinely new information to the answer.

This is the core weakness of top-k in isolation: **it optimizes purely for relevance, and pays no attention to diversity or coverage.** A user asking a broad question ("what are the risks associated with this investment product?") deserves chunks that cover *different* risks, not five near-identical restatements of the single most prominent one.

---

## 14.3 MMR: Maximal Marginal Relevance

**MMR (Maximal Marginal Relevance)** is a re-selection strategy designed directly to fix the redundancy problem. Instead of picking the top k chunks purely by similarity to the query, MMR picks chunks one at a time, balancing two competing goals:

1. **Relevance** — how similar is this candidate chunk to the query?
2. **Novelty** — how *different* is this candidate chunk from chunks already selected?

> MMR selects each next chunk to maximize a weighted combination of "relevant to the query" and "not redundant with what I've already picked" — trading a small amount of pure relevance for a meaningful gain in coverage.

Conceptually, the algorithm works like this: start with a larger candidate pool (say, the top 20 chunks by similarity), then greedily build the final set of k chunks by repeatedly picking whichever remaining candidate scores best on relevance *minus* similarity to already-chosen chunks. A tunable parameter (often called lambda) controls the balance — lambda close to 1 behaves almost like plain top-k, while a lower lambda pushes harder toward diversity.

The tradeoff is intuitive: MMR sometimes leaves a genuinely relevant chunk out in favor of a slightly less relevant but more novel one. For narrow, highly specific questions with a single correct answer, this can occasionally hurt. For broad, multi-faceted questions, it usually helps a lot — which is why MMR is often applied selectively rather than universally, guided by the type of query being handled.

---

## 14.4 Multi-Query Retrieval

MMR diversifies *after* retrieval, by reshuffling which chunks get selected from a single search. **Multi-query retrieval** diversifies *before* retrieval, by changing the search itself.

The idea: instead of running one retrieval pass with one query embedding, generate several *variants* of the query — different phrasings, different angles, different levels of specificity — run retrieval separately for each variant, and then merge and deduplicate the results.

For example, given "How do I reduce my cloud costs?", a multi-query step (usually an LLM call, similar in spirit to the expansion technique from Chapter 13) might generate:

- "How do I reduce my cloud costs?"
- "What are strategies for lowering cloud infrastructure spending?"
- "Ways to optimize compute and storage costs in the cloud"
- "Cloud cost reduction best practices"

Each variant is embedded and searched independently, and the resulting chunk sets are merged — typically deduplicated by chunk ID, and sometimes re-scored using something like reciprocal rank fusion, which rewards chunks that show up near the top across *multiple* query variants.

Why does this help? A single query embedding is one point in vector space, and it can only be "close" to a limited neighborhood of documents. Different phrasings of the same underlying need land in slightly different neighborhoods, and relevant documents may only be close to *some* of those phrasings, not all of them. Multi-query retrieval effectively casts a wider net across the same information need, at the direct cost of running k times as many retrieval calls — a cost that's usually cheap for the vector search itself, but adds up if each variant also requires its own LLM-based rewriting or reranking step downstream.

Multi-query retrieval and query expansion (Chapter 13) are close cousins — the difference is mostly about *where* the extra terms/queries are generated and used. Expansion typically enriches a single query; multi-query retrieval runs genuinely separate searches and merges the results.

---

## 14.5 Parent-Document Retrieval

The last strategy in this chapter addresses a tension that runs through the entire book: **the best chunk size for matching is not the best chunk size for generation.**

Recall from Chapter 6 that small chunks tend to produce more precise embeddings — a 200-token chunk about "the cancellation fee for annual plans" embeds more sharply and matches more precisely than a 2,000-token chunk that covers cancellation fees, billing cycles, and refund timelines all at once, whose embedding is a blurrier average of everything it contains. So for the *matching* step, smaller is usually better.

But for the *generation* step, that same small chunk is often too little context. A 200-token snippet about a cancellation fee might not mention which plans it applies to, or under what conditions — details that live in the surrounding paragraphs, cut off by the chunk boundary.

**Parent-document retrieval** resolves this tension by decoupling what you search against from what you return:

> Index small, precise child chunks for matching — but when a child chunk is retrieved, return its larger parent (the full section, page, or document it belongs to) to the generation step, rather than the child chunk itself.

```
Document
  └── Parent chunk (e.g., full section, ~1500 tokens)
        ├── Child chunk 1 (~200 tokens) ──┐
        ├── Child chunk 2 (~200 tokens)   ├── indexed & searched individually
        └── Child chunk 3 (~200 tokens) ──┘

Query matches Child chunk 2
      │
      ▼
Return the Parent chunk (not just Child chunk 2) to the generator
```

This "search small, return big" pattern requires maintaining a mapping between child chunks and their parents — typically metadata stored alongside each child chunk (an area that overlaps with the metadata strategies in Chapter 7) — and a retrieval step that, after finding matching children, fetches the corresponding parents (deduplicating, since multiple children of the same parent may match) before handing anything to the LLM.

The benefit is real: you get precise matching and rich generation context simultaneously. The cost is added system complexity (you're now managing two levels of chunking and a lookup between them) and larger prompts, since parent chunks consume more of the context window (Chapter 18) than the child chunks that surfaced them.

---

## 14.6 Comparing the Strategies

| Strategy | Solves | Adds |
|---|---|---|
| **Top-k** (baseline) | Basic relevant-chunk retrieval | — |
| **MMR** | Redundant, near-duplicate results | A diversity/relevance tradeoff parameter |
| **Multi-query** | Single-phrasing blind spots | Extra retrieval passes + merge/dedup logic |
| **Parent-document** | Precise matching vs. sufficient context tension | A parent/child mapping + larger prompts |

These strategies compose. A common production pattern is multi-query retrieval feeding into MMR-based selection, over an index built with parent-document chunking — each layer solving a distinct problem, at the cost of a more complex pipeline to build, tune, and debug.

---

## 14.7 A Note of Honesty: More Retrieval Machinery Isn't Always Better

It's tempting to treat this chapter as a checklist — add MMR, add multi-query, add parent-document retrieval, and assume the system only gets better. In practice, each of these adds real cost and real failure surface:

- **MMR can hurt narrow queries.** If there really is one correct chunk and four distractors, pushing for diversity can push the right chunk out in favor of "different but less useful" ones.
- **Multi-query retrieval multiplies latency and cost**, and a poorly generated query variant can pull in irrelevant chunks that a re-ranker (Chapter 15) then has to filter back out.
- **Parent-document retrieval bloats prompts.** Returning full sections for every match can push relevant content out of the context window, or dilute it with irrelevant surrounding material the small chunk-level match didn't actually need.
- **None of these strategies fix a bad index or a bad query.** They operate on top of the embeddings and chunks Part II and III produced — garbage in at those layers is still garbage out here, just diversified or repackaged.

The right strategy — or combination — depends on your query patterns, your document structure, and your latency budget, and is something you should measure with the retrieval metrics in Chapter 27 rather than assume.

---

## 14.8 Chapter Summary

- **Top-k retrieval** is the baseline: return the k chunks most similar to the query embedding. Its main weakness is redundancy — it can return several near-duplicate chunks that say the same thing.
- **MMR (Maximal Marginal Relevance)** re-selects chunks to balance relevance against novelty, reducing redundancy at the cost of occasionally dropping a highly relevant but similar chunk.
- **Multi-query retrieval** generates several phrasings of the query, retrieves for each independently, and merges the results — casting a wider net across a single information need at the cost of extra retrieval passes.
- **Parent-document retrieval** indexes small, precise chunks for matching but returns their larger parent chunk for generation — the "search small, return big" pattern — resolving the tension between precise matching and sufficient context.
- These strategies are complementary and often combined, but each adds latency, cost, and complexity, and none of them compensates for a poorly built index or a poorly formed query.
- Choosing among them should be guided by measured retrieval quality (Chapter 27), not assumed by default.

**Coming up next (Chapter 15):** retrieval gets us a candidate set of chunks, but "similar enough to retrieve" and "actually the best chunk to answer with" are not the same thing. Chapter 15 covers re-ranking — using slower, more accurate models to reorder a retrieved shortlist for precision.
