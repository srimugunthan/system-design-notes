# Chapter 10: Vector Databases

## 10.1 What This Chapter Covers

In Chapter 9, we turned text into vectors — points on a meaning map. But a pile of vectors sitting in a file isn't a retrieval system. You need something that can store millions (or billions) of these vectors, and, given a new query vector, quickly find the handful that are closest to it. That "something" is a **vector database**.

This chapter answers a practical question: **what does a vector database actually do under the hood, and how should you go about choosing one?** We won't crown a single winner — the right choice depends heavily on your scale, team, and constraints — but we'll build a clear mental model of what these systems do and a checklist for evaluating them.

---

## 10.2 What a Vector Database Actually Does

Picture a librarian who isn't just holding books, but holding a map of *where every book sits in meaning-space* (our map from Chapter 9). When you hand this librarian a new query, they don't reread every book — they look at where your query lands on the map and walk straight to the nearest neighborhood.

A vector database is the system that makes this possible at scale. At minimum, it needs to do three things well:

1. **Store** large numbers of vectors (often alongside the original text chunk and metadata like source document, timestamp, or access permissions)
2. **Index** those vectors in a structure that makes similarity search fast, rather than requiring a full scan of every vector for every query (the *how* of this is Chapter 11's entire subject)
3. **Search** by taking a query vector and returning the top-K most similar stored vectors, typically ranked by a distance metric like cosine similarity or dot product

> **Vector database (working definition):** A system purpose-built to store high-dimensional vectors and efficiently retrieve the ones most similar to a given query vector, typically alongside the metadata needed to filter and interpret results.

Most production vector databases also support **metadata filtering** — narrowing a search to only vectors matching certain conditions ("only documents from the last 90 days," "only chunks the current user is permitted to see") — and increasingly, **hybrid search**, blending vector similarity with keyword-style matching (Chapter 12). These two capabilities turn out to matter enormously in real deployments, often more than raw search speed.

---

## 10.3 The Landscape: Categories, Not Just Products

Vector database options generally fall into a few categories. It helps to think in categories first, then evaluate specific products within the category that fits your situation.

| Category | Examples | General Character |
|---|---|---|
| **Standalone library, embedded in your app** | FAISS | Not a full database — a fast similarity-search library you embed directly in your application; you handle persistence, scaling, and metadata yourself |
| **Purpose-built vector database (self-hosted)** | Milvus, Weaviate (self-hosted mode) | Full database features — persistence, filtering, clustering — that you deploy and operate yourself |
| **Purpose-built vector database (managed/cloud)** | Pinecone, Weaviate Cloud, Milvus-as-a-service | Same capabilities, operated for you; you trade operational control for convenience |
| **Vector search bolted onto an existing database** | pgvector (PostgreSQL extension) | Adds vector search to a database you may already run, trading some specialized performance for operational simplicity and one fewer system to manage |

It's worth being precise about what each category is actually good at rather than treating this as a leaderboard:

- **FAISS** is best understood as a fast, well-tested *algorithm library* (it implements many of the indexing algorithms we cover in Chapter 11) rather than a database. It's a strong choice when you want tight control, are comfortable building the surrounding infrastructure yourself, and don't need built-in metadata filtering, multi-tenancy, or persistence out of the box.
- **Milvus and Weaviate** are purpose-built vector databases with the operational features (replication, filtering, hybrid search, access control) that FAISS leaves to you. They can be self-hosted or consumed as managed services, giving you a choice along the control-vs-convenience spectrum.
- **Pinecone** is a fully managed vector database — you don't operate any infrastructure yourself, which is attractive for teams that want to move fast and not think about index maintenance, sharding, or scaling.
- **pgvector** turns PostgreSQL into a vector store. Its main appeal isn't raw search performance — it's that if you already run Postgres for your application data, you can add vector search without introducing an entirely new system to operate, back up, and secure.

None of these categories is universally "the best" — they represent different points on a control-vs-convenience and specialization-vs-simplicity spectrum, and the right pick depends on the criteria in the next section.

---

## 10.4 Selection Criteria: What to Actually Weigh

### 10.4.1 Self-Hosted vs. Managed

This is usually the first fork in the road, and it mirrors the open-source vs. proprietary tradeoff from Chapter 9.

| | Self-Hosted | Managed |
|---|---|---|
| **Operational burden** | You handle scaling, upgrades, backups, uptime | Vendor handles it |
| **Cost model** | Infrastructure cost, often cheaper at very high scale | Usage-based pricing, can grow expensive at scale |
| **Data residency / privacy** | Full control — data never leaves your environment | Data lives with a third party |
| **Time to production** | Slower — real infrastructure work required | Faster — often production-ready in a day |

A team in a regulated industry with strict data residency requirements (see Chapter 35's discussion of financial services constraints) will often lean self-hosted almost by default. A small team validating a product idea will often reasonably prefer managed, and revisit the decision once scale or cost justifies the switch.

### 10.4.2 Scale Requirements

"Scale" here has at least two dimensions worth separating:

- **Corpus size** — thousands of chunks behaves very differently from hundreds of millions. Small corpora can often get away with a simple, even brute-force setup (see Chapter 11); large corpora need serious indexing and sharding support.
- **Query throughput and latency needs** — a small internal tool answering a handful of queries per minute has very different requirements than a customer-facing product serving thousands of queries per second with sub-100ms latency budgets.

Be honest about which regime you're actually in before optimizing for a scale you don't yet have. Over-engineering for hypothetical billion-vector scale when you have 50,000 documents adds operational complexity for no real benefit.

### 10.4.3 Hybrid Search Support

As Chapter 12 will make the case for in detail, pure dense vector search alone often isn't enough — you frequently need to blend in exact keyword matching. Some vector databases support this natively (combined dense + sparse indexing, built-in fusion of results); others require you to bolt on a separate keyword search system (like Elasticsearch or OpenSearch) and merge results yourself in application code. If hybrid search is a known requirement rather than a "maybe later," it's worth weighing heavily during selection rather than discovering the gap after you've already committed.

### 10.4.4 Metadata Filtering Support

Real-world retrieval is rarely "just" semantic search — it's semantic search *constrained* by business logic: "only search documents this user has access to," "only search the current tenant's data in a multi-tenant system," "only search documents published in the last year." How well and how efficiently a database supports combining vector similarity with these filters varies significantly. A database that has to filter *after* retrieving nearest neighbors — rather than filtering *during* the search — can silently return far fewer relevant results than expected once filters are applied, a subtle failure mode worth explicitly testing for before you commit to a system.

### 10.4.5 Ecosystem and Integration Fit

Finally, weigh how well a candidate fits the system you already have:

- Does it have solid client libraries in your team's language and mature integration with your orchestration framework?
- Does it fit naturally with your existing data infrastructure (e.g., you already run Postgres, so pgvector removes an entire new system to operate)?
- Does it have the observability hooks you'll need in production (Chapter 32) — query latency, index health, memory usage?
- How mature is the community and documentation, especially for debugging edge cases at 2 a.m.?

The "best" vector database on paper that doesn't fit your team's existing stack often loses, in practice, to a "good enough" one that integrates cleanly.

---

## 10.5 A Note of Honesty: There Is No Universally Correct Choice

It's tempting to want a definitive ranking — "use X, it's the best." Resist that instinct. Vector database benchmarks are notoriously sensitive to the specific dataset, vector dimensionality, hardware, index configuration, and query pattern used to produce them; a benchmark showing one system dramatically outperforming another under one set of conditions can flip entirely under different conditions. Treat published benchmarks as a starting hypothesis to validate on your own workload, not a verdict to trust blindly.

It's also worth acknowledging that this decision is not permanent, but it isn't free to change either — migrating a production vector database, especially one holding hundreds of millions of vectors with live traffic, is a real engineering project, not a config change. Choosing conservatively for your near-term (12–18 month) needs, rather than optimizing for a hypothetical future scale, is usually the more pragmatic path.

---

## 10.6 Chapter Summary

- A **vector database** stores vectors alongside their metadata and enables fast similarity search — retrieving the nearest neighbors to a query vector rather than scanning every stored vector.
- Options generally fall into categories: **embedded libraries** (FAISS), **self-hosted purpose-built databases** (Milvus, Weaviate), **managed vector databases** (Pinecone, Weaviate Cloud), and **vector extensions to existing databases** (pgvector).
- **Self-hosted vs. managed** is usually the first major decision, trading operational control and data residency against speed of deployment and reduced operational burden.
- **Scale requirements** — both corpus size and query throughput/latency needs — should be assessed honestly, not over-engineered for hypothetical future scale.
- **Hybrid search support** and **metadata filtering support** are frequently more decisive in practice than raw search speed, especially for real-world, permission-aware, filtered retrieval.
- **Ecosystem and integration fit** with your team's existing stack often outweighs marginal performance differences between systems.
- Published benchmarks are highly sensitive to configuration and workload — validate on your own data rather than trusting rankings at face value.
- There is no single "best" vector database — only the one that best fits your scale, constraints, and team.

**Coming up next (Chapter 11):** we've covered *where* vectors get stored — now we'll go one level deeper into *how* a vector database finds nearest neighbors so quickly, covering the indexing algorithms — HNSW, IVF, and product quantization — that make approximate search fast at scale.
