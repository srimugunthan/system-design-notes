# Chapter 11: Indexing Algorithms

## 11.1 What This Chapter Covers

In Chapter 10, we treated the vector database as something of a black box: you put vectors in, you get nearest neighbors out. This chapter opens that box. **How does a system search through millions — or billions — of vectors and return the closest matches in milliseconds, without comparing the query to every single one?**

The answer is a family of clever data structures and algorithms broadly called **Approximate Nearest Neighbor (ANN)** search. We'll build intuition for why exact search doesn't scale, and then walk through the three ideas that show up again and again in real systems: HNSW, IVF, and product quantization.

---

## 11.2 Why Exact Nearest-Neighbor Search Doesn't Scale

The most obvious way to find the nearest neighbors to a query vector is **brute force**: compare the query against every single stored vector, compute a distance for each one, and sort to find the closest matches. This is called **exact nearest-neighbor search**, because it's mathematically guaranteed to find the true closest vectors — no approximation, no shortcuts.

The problem is simple arithmetic. If you have 10 million chunks in your knowledge base and a user submits a query, brute-force search requires 10 million distance calculations — for that single query. Now imagine thousands of queries per minute, each one requiring millions of comparisons, each comparison touching every dimension of a vector that might have hundreds or thousands of numbers in it. This does not scale gracefully. Latency grows directly with corpus size — double your documents, and (roughly) double your search time. For a knowledge base that might grow to tens or hundreds of millions of chunks over a system's lifetime, brute force quickly becomes unusable for anything resembling real-time search.

> **Core idea:** Approximate Nearest Neighbor (ANN) search accepts a small, controlled chance of missing the *absolute* closest match, in exchange for search that stays fast even as the corpus grows into the millions or billions.

This trade is almost always worth it in RAG systems. If the true best-matching chunk is #1 and an ANN index instead returns it as the #3 result (with two nearly-as-relevant chunks ahead of it), that's rarely the difference between a good and a bad answer — especially once re-ranking (Chapter 15) further refines the candidate list. What *would* be unacceptable is a search that takes ten seconds per query. ANN algorithms exist to make that tradeoff deliberately and tunably, rather than accepting it as an accident of scale.

---

## 11.3 HNSW: Hopping Through a Social Network

**HNSW (Hierarchical Navigable Small World)** is one of the most widely used ANN algorithms today, and its core idea has a very intuitive analogy: think about how you'd actually find a specific person in a huge city if you only knew a few people yourself.

You wouldn't check every resident one by one. You'd start with someone you know, ask "who do you know that's closer to who I'm looking for?", hop to that next person, and repeat — each hop getting you closer, until you land on (or very near) the person you wanted. This is roughly how HNSW works, except the "people" are vectors and the "friendships" are edges in a graph built at indexing time.

Concretely, HNSW builds a **multi-layer graph** where each vector is a node connected to a handful of its nearest neighbors:

- The **top layer** is sparse, with long-range connections — like knowing a few well-connected people who each know completely different circles, useful for covering large distances quickly.
- **Lower layers** get progressively denser, with short-range connections — like your close friend group, useful for fine-grained final hops.
- A search starts at the top layer, greedily hops toward the query, then drops down a layer and repeats, refining its position at each level until it reaches the bottom layer and has a strong candidate set.

This structure means a search touches a tiny fraction of the total vectors in the index — closer to logarithmic growth with corpus size than linear — which is exactly why HNSW stays fast even as a corpus grows very large. The cost is that the graph itself takes memory to store, and building it (all those "friendships") takes real time and compute upfront, at insertion time. HNSW is generally regarded as offering some of the best speed-versus-accuracy tradeoffs among ANN methods, which is why it's a common default in modern vector databases — but that graph structure living in memory is a real resource cost worth planning for at scale (Chapter 30 covers this scaling math further).

---

## 11.4 IVF: A Table of Contents for Your Vectors

**IVF (Inverted File Index)** takes a completely different approach, and it maps neatly onto something familiar: a book's table of contents, or a library's section signage.

Before you search a library shelf by shelf, you first walk to the right *section* — "Fiction," "History," "Science" — and only then start browsing individual books. IVF does the vector-space equivalent of this:

1. During indexing, IVF runs a clustering step over the full set of vectors, grouping them into a fixed number of **buckets** (technically, clusters around learned centroids — but think of them simply as neighborhoods on the meaning map from Chapter 9).
2. Each stored vector is assigned to its nearest bucket, the way a book gets shelved in one section.
3. At query time, instead of comparing the query against every vector, IVF first compares the query only against the small number of bucket centroids, identifies the few buckets closest to the query, and then only searches *within* those buckets.

This collapses the search space dramatically — instead of scanning millions of vectors, you might scan only the few thousand that live in the handful of most relevant buckets. The main tuning knob is how many buckets to search (often called `nprobe`): search only 1 bucket and you're extremely fast but risk missing relevant vectors that landed in a neighboring bucket; search many buckets and you approach exhaustive-search accuracy at exhaustive-search cost. IVF is often paired with product quantization (next section) to compress what's stored inside each bucket, and that combination is a very common real-world configuration.

---

## 11.5 Product Quantization: Compressing the Map

HNSW and IVF both solve the problem of *how many* vectors you compare against. **Product quantization (PQ)** solves a different, complementary problem: *how much memory each vector takes up*.

A useful analogy: imagine trying to describe a person's face with perfect photographic precision versus describing it with a rough composite sketch built from a small set of standard facial features — "this nose shape, that eye shape, this jaw shape." The sketch loses detail, but it takes vastly less information to store and is often *good enough* to recognize the person.

Product quantization does something similar to vectors:

1. It splits each high-dimensional vector into several smaller sub-vectors (chunks of the original vector).
2. For each sub-vector position, it builds a small "codebook" of representative patterns, learned from the data — analogous to a standard set of nose shapes or eye shapes.
3. Each sub-vector is then replaced with a compact code pointing to its closest representative pattern in the codebook, rather than storing the original, full-precision numbers.

The result is a dramatic reduction in memory footprint — often storing a compressed vector in a small fraction of the space its original floating-point form required — at the cost of some precision, since the compressed vector is now an approximation of an approximation. This matters enormously at scale: an index holding hundreds of millions of full-precision vectors might not even fit in memory on a single machine, while a product-quantized version of the same index often will. PQ is rarely used alone; it's typically layered on top of IVF (compress what's inside each bucket) or combined with graph methods, giving you a way to trade a further slice of accuracy for a large reduction in memory cost.

---

## 11.6 Putting It Together: A Comparison

No single algorithm is uniformly best — each makes a different trade among speed, memory, and accuracy, and real systems often combine them.

| Approach | Core Idea | Speed | Memory | Accuracy | Best Fit |
|---|---|---|---|---|---|
| **Brute force (exact)** | Compare against every vector | Slow at scale | High (full vectors) | Perfect (exact) | Small corpora, or as a correctness baseline |
| **HNSW** | Navigable multi-layer graph | Very fast | Higher (graph structure adds overhead) | Very high | General-purpose default at moderate-to-large scale |
| **IVF** | Cluster into buckets, search nearest buckets | Fast (tunable via bucket count) | Moderate | Good, tunable | Very large corpora where build simplicity matters |
| **Product Quantization (PQ)** | Compress vectors into compact codes | Fast (often paired with IVF) | Very low | Reduced (approximation of an approximation) | Memory-constrained, very large-scale deployments |
| **IVF + PQ (combined)** | Bucket first, then compress | Fast | Very low | Moderate, tunable | Billion-scale indexes where memory is the binding constraint |

The right choice depends on where your actual bottleneck is. If your corpus comfortably fits in memory and query latency is the priority, HNSW alone is a strong, common default. If memory is the binding constraint — because the vector count is enormous or the hardware budget is tight — IVF combined with PQ becomes far more attractive, even though it costs some accuracy. Most managed vector databases (Chapter 10) let you pick or tune these under the hood rather than requiring you to implement them yourself, but understanding what's happening beneath that configuration option is what lets you tune it sensibly instead of guessing.

---

## 11.7 A Note of Honesty: Approximate Means Approximate

It's worth being direct about what "approximate" really costs you. ANN indexes can, and occasionally will, miss the truly best-matching chunk for a query — that's the entire premise of the trade. In most RAG use cases this is a fine trade, since retrieval typically returns several candidates and generation (Part V) or re-ranking (Chapter 15) can compensate for an imperfect ranking. But it stops being fine in situations where a single, specific piece of information absolutely must be found — and in those cases, the fix usually isn't abandoning ANN search altogether, but tightening its accuracy knobs (more HNSW connections, more IVF buckets probed, less aggressive quantization) at the cost of some speed and memory, or layering a hybrid keyword pass (Chapter 12) that catches exact matches ANN indexes are prone to blur.

It's also worth remembering that these indexing choices interact with the embedding model itself (Chapter 9) — the index doesn't know or care what the vectors mean, it only operates on their geometry. An index can be built and tuned perfectly and still deliver poor retrieval quality if the underlying embeddings weren't a good fit for the content in the first place. Indexing algorithms make a good embedding fast to search; they don't make a bad embedding good.

---

## 11.8 Chapter Summary

- **Brute-force exact nearest-neighbor search** compares a query against every stored vector, and its cost grows directly with corpus size — impractical for large-scale RAG systems.
- **Approximate Nearest Neighbor (ANN)** search deliberately trades a small, tunable amount of accuracy for large gains in search speed.
- **HNSW** builds a multi-layer navigable graph and searches by hopping toward the query through progressively finer layers, similar to navigating a social network — fast and accurate, at the cost of memory for the graph structure.
- **IVF** clusters vectors into buckets ahead of time and searches only the buckets nearest the query, like using a table of contents instead of scanning every page.
- **Product quantization (PQ)** compresses vectors into compact codes built from learned representative patterns, dramatically reducing memory at some cost to precision.
- These techniques are frequently **combined** (e.g., IVF + PQ) to balance speed, memory, and accuracy for a given deployment's constraints.
- There is no universally best algorithm — the right choice depends on whether your bottleneck is speed, memory, or accuracy, and most vector databases let you tune these settings rather than pick a fixed algorithm blindly.
- Indexing algorithms make a good embedding searchable at scale — they cannot fix a poorly chosen embedding model.

**Coming up next (Chapter 12):** we've now covered how dense vector search finds semantically similar chunks efficiently — next we'll bring keyword-based search back into the picture, covering BM25 and hybrid retrieval, and show how combining exact keyword matching with semantic search closes the gaps either approach leaves on its own.
