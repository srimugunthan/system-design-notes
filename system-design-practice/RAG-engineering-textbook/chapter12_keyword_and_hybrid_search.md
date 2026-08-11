# Chapter 12: Keyword & Hybrid Search

## 12.1 What This Chapter Covers

Across the last three chapters, we've built a fast, scalable pipeline for finding chunks that are *semantically* similar to a query. It's tempting to think that solves retrieval. It doesn't. This chapter answers a question practitioners often learn the hard way: **why does pure semantic search fail on exact-match queries, and what do we do about it?**

We'll look at classic keyword search (specifically **BM25**), understand exactly where dense embeddings fall short, and then cover **hybrid search** and **reciprocal rank fusion (RRF)** — the standard toolkit for combining lexical and semantic retrieval into something more reliable than either alone.

---

## 12.2 Where Dense Search Quietly Fails

Recall from Chapter 9 that dense embeddings blend meaning across an entire vector — the concept of "a small animal resting on a surface" gets smeared across hundreds of dimensions rather than tied to specific words. That's exactly what makes dense search good at paraphrase and intent matching. It's also exactly what makes it unreliable for a specific, common category of query: ones where the *exact string* is the whole point.

Consider these query types:

- **Product/part codes**: "SKU-48291-B" — a dense model has likely never seen this exact string during training and has no reliable way to represent it distinctly from other similar-looking codes.
- **Acronyms**: "GDPR" vs. "CCPA" — semantically these are both "data privacy regulations," and a dense embedding may place them close together, exactly when you needed the search to distinguish them precisely.
- **Names**: "Sarah Chen" vs. "Sara Chen" vs. a different "Sarah Chen" entirely — dense embeddings capture rough semantic neighborhoods, not fine string-level distinctions.
- **Error codes / IDs**: "ERR_5003" — often near-meaningless as language, but critically important as an exact token to match.

In each case, the dense model does what it was trained to do — represent meaning — but the query wasn't really about meaning. It was about finding *this exact thing*, not *things like it*. This is the blind spot we flagged back in Chapter 9, and it's the reason pure semantic search, on its own, is not a complete retrieval strategy for most real systems.

The reverse failure is just as real, and it's the reason pure keyword search alone isn't a complete answer either: a keyword-only system asked "how do I get my money back?" will not reliably retrieve a document titled "Refund Policy" unless that document happens to contain the specific word "money" — even though any human would immediately recognize the connection. Keyword search has no concept of synonyms or paraphrase; it only knows whether a term is literally present.

> **Core idea:** Dense search finds things that *mean* the same; keyword search finds things that *say* the same. Real queries need both, because you rarely know in advance which kind of match a given query actually needs.

---

## 12.3 BM25: Classic Keyword Search, Conceptually

**BM25 (Best Matching 25)** is the dominant algorithm behind traditional keyword — or **lexical** — search, and it has remained a strong, hard-to-beat baseline for decades despite the rise of neural approaches. You don't need its formula to understand what it's doing; two intuitive ideas carry almost all of the weight.

**Idea 1 — Term frequency (but with diminishing returns).** If a document mentions "refund" five times, that's a decent signal it's actually about refunds, more so than a document that mentions it once. But BM25 doesn't treat the tenth mention as ten times more significant than the first — the scoring curve flattens out, so a document that happens to repeat a word obsessively doesn't unfairly dominate every result. Frequency matters, but with common sense limits on how much.

**Idea 2 — Inverse document frequency (rare terms carry more weight).** If a query contains the word "the," that tells you almost nothing — "the" appears in nearly every document, so its presence doesn't help distinguish a good match from a bad one. But if a query contains "arbitration," a word that appears in only a handful of documents in your whole corpus, finding that word is a strong, specific signal. BM25 automatically down-weights common words and up-weights rare ones, without anyone needing to hand-build a stop-word list.

Put together, BM25 essentially asks: *"Which documents contain the query's rare, distinctive terms, repeated a meaningful (but not suspiciously excessive) number of times?"* It also typically normalizes for document length, so a long document doesn't win purely by having more words to potentially match against.

BM25 is the direct conceptual ancestor of the sparse embeddings we introduced in Chapter 9 — modern sparse embedding techniques are, in large part, attempts to capture the same term-frequency-and-rarity intuition in a learned, vector-shaped form. If you understand BM25, you already understand most of what sparse embeddings are doing under the hood.

BM25's strengths map directly onto dense search's weaknesses: it matches exact terms reliably, treats rare terms (like codes, names, and acronyms) as strong signals almost by construction, and requires no training or embedding model at all — just the raw text. Its weakness is equally direct: it has no notion of meaning. "Automobile" and "car" are, to BM25, two completely unrelated tokens.

---

## 12.4 Hybrid Search: Running Both, Together

Given that dense and lexical search fail in complementary places, the practical fix is straightforward in concept: **run both retrieval methods on the same query, and combine their results.** This is **hybrid search**.

A typical hybrid pipeline looks like this:

```
User Query
     │
     ├──► Dense retriever (embedding similarity) ──► Ranked list A
     │
     └──► Lexical retriever (BM25 / sparse)       ──► Ranked list B
     │
     ▼
 Fusion step ──► single merged, re-ranked result list
```

The dense retriever finds chunks that are conceptually related even without shared words. The lexical retriever finds chunks that contain the exact rare terms in the query, even if the phrasing is completely different from how the document expresses the idea elsewhere. Neither retriever needs to be "right" on its own — the value comes from covering each other's blind spots. A query like "what's the penalty under GDPR Article 17" benefits from lexical search nailing "GDPR Article 17" precisely, while dense search helps surface a document phrased as "consequences of violating the right to erasure" that never uses the word "penalty" at all.

As covered in Chapter 10, some vector databases support hybrid search natively, running both retrieval modes internally and returning fused results; others require you to query a vector store and a keyword-search system (like Elasticsearch or OpenSearch) separately and merge the two result lists yourself in application code. Either way, the interesting engineering problem is the same: **how do you actually combine two ranked lists into one?**

---

## 12.5 The Fusion Problem: Why You Can't Just Add Scores

The naive approach is to take the dense similarity score and the BM25 score for each chunk and add them together, or average them. This runs into a real problem: **the two scores aren't on comparable scales.** A cosine similarity score typically lives in a tight, bounded range; a BM25 score is theoretically unbounded and varies wildly depending on term rarity and document length in ways that differ from corpus to corpus. Averaging a 0.83 cosine similarity with a 14.2 BM25 score produces a number that doesn't mean anything coherent — and worse, that meaninglessness can shift depending on the specific query, making any fixed weighting brittle in practice.

This is exactly the problem **reciprocal rank fusion (RRF)** was designed to sidestep.

---

## 12.6 Reciprocal Rank Fusion (RRF)

RRF makes a simple but effective move: **ignore the raw scores entirely, and use only each result's *rank position* within its own list.** Think of it like combining two independent "best restaurants in town" lists from two different critics who use completely different rating scales (one rates out of 5 stars, one rates out of 100 points) — you can't meaningfully average "4.5 stars" with "82 points," but you absolutely can compare "this restaurant was #1 on critic A's list" with "this restaurant was #3 on critic B's list," because rank position means the same thing regardless of the underlying scale.

For each chunk, RRF computes a fusion score based on where that chunk ranked in each individual result list — roughly, a value that rewards being ranked highly, with rapidly diminishing rewards for lower ranks (so being #1 matters far more than the difference between #40 and #41). A chunk's final RRF score is the sum of this rank-based value across every list it appeared in. A chunk that ranked highly in *both* the dense and lexical lists gets a strong combined score; a chunk that ranked highly in only one list still gets credit, just less of it; a chunk that appeared in neither list gets nothing.

> **Reciprocal rank fusion (core idea):** Combine multiple ranked lists by rewarding rank position rather than raw score, so that lists using incompatible scoring scales can still be merged fairly and robustly.

RRF's popularity comes from its practicality: it requires no score normalization, no calibration between retrievers, and no per-corpus tuning to work reasonably well — you can add a third or fourth retriever (say, a dedicated code-search index, discussed further in Chapter 37) to the fusion step without redesigning anything. It's not the theoretically optimal fusion method in every scenario, and more sophisticated learned fusion approaches exist, but as a robust, low-maintenance default, it's hard to beat, which is why it shows up so often as the fusion step inside hybrid search implementations in practice.

---

## 12.7 A Note of Honesty: Hybrid Search Isn't Free

Hybrid search solves a real and common failure mode, but it isn't a free upgrade:

- **It roughly doubles retrieval infrastructure and latency** — you're now running two searches per query (or maintaining two indexes) instead of one, and fusing the results, which adds both engineering surface area and response time.
- **It doesn't fix a bad chunking strategy or a poorly matched embedding model** — hybrid search widens what can be found, but Chapter 6's chunking decisions and Chapter 9's embedding choice still bound what's findable in the first place.
- **RRF is a robust default, not a guarantee of optimal ranking** — for some query distributions, a carefully tuned weighted combination (once you've done the work to calibrate it) can outperform RRF; the tradeoff is that tuning is fragile and needs ongoing maintenance as your data changes, which is exactly the maintenance burden RRF exists to avoid.
- **More retrieved candidates isn't automatically better** — fusing two lists means the generation step (Part V) may now receive a broader, noisier set of candidates, which raises the importance of the re-ranking step we'll cover next in Chapter 15 to sort real signal from near-miss noise before it ever reaches the language model.

Hybrid search is close to a default best practice in modern RAG systems for good reason, but it's worth being clear-eyed that it's solving a specific, well-understood gap — not delivering some universal, free improvement to retrieval quality.

---

## 12.8 Chapter Summary

- Pure **dense (semantic) search** fails on exact-match needs — product codes, acronyms, names, and IDs can be blurred together because meaning is blended across the whole embedding vector.
- Pure **keyword (lexical) search** misses paraphrase and synonyms — it only knows whether a term is literally present, not whether two different terms mean the same thing.
- **BM25** scores documents using two intuitive signals: **term frequency** (with diminishing returns) and **inverse document frequency** (rare terms carry more weight than common ones), with no training required.
- BM25 is the conceptual ancestor of the **sparse embeddings** introduced in Chapter 9.
- **Hybrid search** runs dense and lexical retrieval together and fuses their results, covering each method's blind spots with the other.
- Raw scores from dense and lexical retrievers **aren't on comparable scales**, so naively adding or averaging them produces meaningless, unstable results.
- **Reciprocal rank fusion (RRF)** merges ranked lists using each result's rank position rather than its raw score, making it robust to incompatible scoring scales and easy to extend to more retrievers.
- Hybrid search adds real infrastructure and latency cost, doesn't fix upstream chunking or embedding choices, and increases the importance of a strong re-ranking step downstream.

**Coming up next (Chapter 13):** with Part III's storage and retrieval mechanics in place, Part IV turns to what happens *before* retrieval even runs — starting with query understanding, where we'll look at how raw user queries get interpreted, rewritten, and expanded before they're ever sent to a retriever.
