# Backend Design: Scaling to Millions of Earnings Reports

## Context

This extends [system-design-Earnings-report-RAG.md](./system-design-Earnings-report-RAG.md) (base architecture) and [system-design-Earnings-report-RAG-chat-interface.md](./system-design-Earnings-report-RAG-chat-interface.md) (UI↔backend interface). Those docs assume the "Production Scale" numbers from the base doc (~100,000s documents, ~1,000,000s chunks). This doc pushes two orders of magnitude further: **millions of earnings reports**, continuously growing (every public company files 10-K/10-Q/8-K every quarter, decades of history, thousands of tickers).

At that scale, the backend stops being "a vector store and an LLM call" and becomes a data platform: an ingestion pipeline that never fully stops, a storage layer that must be sharded and tiered, and a retrieval path that has to stay fast even as the index grows without bound.

## Scale Assumptions

| Dimension | Base doc (production) | This doc (target) |
|---|---|---|
| Documents | ~100,000s | 5-10 million (all US public filers, 20+ years, 10-K/10-Q/8-K) |
| Chunks (avg ~500/doc) | ~1,000,000s | 2-5 billion |
| Vector storage (1536-dim float32) | GBs | ~30-60 TB raw (before compression) |
| Ingestion rate (steady state) | Batch, ad hoc | ~10,000-50,000 new/amended documents/day (earnings season spikes) |
| Query rate | < 500ms latency, low QPS | 1,000+ QPS sustained, spiky around earnings season |
| Metadata cardinality | Handful of companies | ~10,000+ tickers × ~80 quarters × multiple filing types |

The two hard constraints this doc designs around: **ingestion never stops** (new filings and amendments arrive continuously, especially in bursts during earnings season) and **the index is too large for a single node** — both the base doc's local FAISS/Chroma and a single-machine "production scale" setup break down here.

## High-Level Architecture

```
                          ┌────────────────────────────────────────┐
                          │            DATA SOURCES                 │
                          │  SEC EDGAR full-text/bulk feed, filer    │
                          │  uploads, scraping, XBRL structured data │
                          └───────────────────┬──────────────────────┘
                                               ▼
                          ┌────────────────────────────────────────┐
                          │        INGESTION QUEUE (Kafka)          │
                          │  partitioned by ticker; dedup by         │
                          │  content hash; replay-able               │
                          └───────────────────┬──────────────────────┘
                                               ▼
                ┌──────────────────────────────────────────────────────────┐
                │                INGESTION WORKER POOL (autoscaled)          │
                │  Load → Parse → Chunk → Embed(batched) → Version → Store   │
                └───────┬───────────────────┬──────────────────┬────────────┘
                        ▼                   ▼                  ▼
             ┌────────────────┐  ┌────────────────────┐  ┌───────────────────┐
             │  Object Store   │  │  Vector Store        │  │  Metadata Store    │
             │  (S3, raw docs, │  │  (sharded, replicated,│  │  (partitioned OLTP,│
             │  tiered by age) │  │  HNSW/IVF-PQ index)   │  │  company/quarter/  │
             │                 │  │                       │  │  version)          │
             └────────────────┘  └──────────┬────────────┘  └─────────┬──────────┘
                                              │                        │
                                              ▼                        ▼
                          ┌────────────────────────────────────────────┐
                          │        QUERY / RETRIEVAL FANOUT LAYER        │
                          │  metadata pre-filter → shard routing →        │
                          │  parallel ANN search → merge → re-rank        │
                          └───────────────────┬───────────────────────────┘
                                               ▼
                          ┌────────────────────────────────────────┐
                          │     RAG WORKER POOL (stateless, per      │
                          │     the chat-interface doc's queue)      │
                          └────────────────────────────────────────┘
```

## 1. Ingestion pipeline: decouple arrival from processing

At millions-of-documents scale, ingestion is not a batch script — it's a continuous stream with bursty load (earnings season concentrates thousands of filings into a few weeks).

- **Kafka (or equivalent) as the ingestion buffer**, partitioned by ticker. This absorbs bursts without dropping work and makes ingestion replay-able (reprocess a bad embedding run without re-scraping sources).
- **Idempotency via content hashing**: every document gets a hash of its normalized content; re-ingesting the same filing (common — EDGAR sometimes republishes) is a no-op, not a duplicate.
- **Amendments and restatements are versions, not overwrites**: an amended 10-K/A must not silently replace the original in a way that breaks point-in-time queries ("what did they report *originally* vs. *after restatement*"). Store `(document_id, version, superseded_by)` in the metadata store rather than mutating in place.

**Tradeoff:** a queue-based pipeline adds latency between "document available" and "document searchable" (seconds to low minutes, depending on worker throughput) compared to a synchronous ingest-and-block call. That's the right tradeoff here — nothing about earnings analysis requires sub-second ingestion freshness, and the alternative (synchronous ingestion) would mean an ingestion burst directly stalls the pipeline that's also serving queries.

## 2. Chunking & embedding at scale

- **Parallelize chunking and embedding across the worker pool**, batched — embedding APIs (and self-hosted embedding servers) are far more throughput-efficient per-batch (e.g., 100 texts/call) than per-chunk calls; at billions of chunks, unbatched calls would be both slow and cost-prohibitive.
- **Embedding provider tradeoff at this scale:**

| Option | Throughput/cost at scale | Control | Verdict |
|---|---|---|---|
| Hosted API (OpenAI, etc.) | Pay-per-token; at billions of chunks this becomes a dominant, recurring cost | No infra to run | Fine for moderate scale or bootstrapping; gets expensive fast at this document count |
| Self-hosted embedding model (sentence-transformers, served via batched inference, e.g. TEI/vLLM) | High throughput, marginal cost ~compute only | Requires GPU fleet, model updates, monitoring | **Better fit at millions-of-documents scale** — the fixed cost of running the fleet amortizes far below per-token API pricing once ingestion volume is this high |

- **Dedup at the chunk-embedding level**: boilerplate legal language (risk-factor sections, standard disclaimers) repeats near-verbatim across thousands of filings. Hash-and-cache identical/near-identical chunk text before embedding — this alone can cut embedding volume substantially, since earnings filings are notoriously repetitive across companies and quarters for a given filer.

**Tradeoff:** self-hosting embeddings trades ongoing infra ownership (GPU capacity planning, model version drift) for a large reduction in marginal cost — worth it once ingestion volume crosses from "occasional" to "continuous, millions-scale." Below that crossover, the hosted API's operational simplicity wins.

## 3. Storage layer: this is where single-node designs break

### Object storage (raw documents)

- Raw filings go to object storage (S3 or equivalent), **tiered by access recency**: recent quarters in standard storage, older filings (5+ years) moved to infrequent-access/archive tiers. Earnings-report queries skew heavily toward recent quarters, so this significantly cuts storage cost with minimal impact on the common case — and archive-tier retrieval latency (seconds) is acceptable for the rare "pull up a 2009 10-K" request.

### Vector store: must be distributed, not a single FAISS/Chroma process

At 2-5 billion chunks, a single-node index doesn't fit in memory and doesn't parallelize query load. This requires a **distributed vector database** (Milvus, Vespa, Weaviate cluster, or a managed service like Pinecone) with:

- **Sharding**, primarily **by time range and/or ticker prefix** rather than pure hash sharding — because most queries filter by company and/or recent quarters (metadata filtering is already core to this system, per the base doc), time/ticker-aligned shards let a filtered query hit a handful of shards instead of fanning out to all of them.
- **Replication** for read throughput (retrieval is read-heavy, 1,000+ QPS target) and availability — each shard has 2-3 replicas; queries load-balance across replicas.

**Index type tradeoff** (this matters enormously at billions-of-vectors scale):

| Index | Recall | Memory | Query latency | Verdict |
|---|---|---|---|---|
| Flat / exact search | 100% | Prohibitive at this scale (O(n) per query) | Unusable past low millions | Not viable here |
| **HNSW** | High (~95-99%) | High — graph structure held largely in memory | Fast | Good for hot/recent shards where recall and latency both matter most |
| **IVF-PQ** (inverted file + product quantization) | Lower (~85-95%, tunable) | Much lower — vectors compressed 8-32x | Fast, slightly slower than HNSW | Good for cold/older shards where a modest recall trade is acceptable and memory cost dominates |

**Tradeoff — tiered indexing by recency:** use HNSW for recent quarters (last 2-3 years, where query volume and recall expectations are highest) and IVF-PQ for older archives (where queries are rarer and a small recall hit is an acceptable trade for the large memory savings). This mirrors the object-storage tiering above and is the single biggest lever for keeping memory/infra cost sane at billions-of-vectors scale, at the cost of slightly lower recall on old-filing queries — an acceptable trade since those are the least common queries.

### Metadata store: partition, don't scale vertically

The metadata store (companies, quarters, filing versions, extracted metrics) needs to support both point lookups (by ticker+quarter) and joins for cross-company comparisons. A single unpartitioned Postgres instance won't hold up at 10,000+ tickers × 80+ quarters × versions.

- **Partition by ticker range or by filing year** in a horizontally-scaled relational store (e.g., Postgres with Citus, or a managed distributed SQL system like CockroachDB/Spanner) — keeps individual partition size bounded and lets ticker-scoped queries (the common case) hit one partition.
- **Tradeoff:** distributed SQL adds operational complexity (cross-partition joins are more expensive, transactions can span partitions) versus a single Postgres box, but a single box hits both storage and connection-throughput ceilings well before this scale — the added complexity is the cost of not having a scaling wall.

## 4. Retrieval fan-out: making sharding invisible to the query path

A user's query becomes: **metadata pre-filter → determine candidate shards → parallel ANN search on each → merge & re-rank → return top-k.**

- **Metadata pre-filtering before the ANN search** (not after) is essential at this scale — filtering by ticker/quarter first narrows the search to 1-3 shards instead of running an approximate search across the full multi-billion-vector index and discarding most results. This is a direct extension of the base doc's `filter={'company': 'AAPL'}` (line 351), just now load-bearing for performance, not only precision.
- **Cross-shard queries** (company comparisons, sector-wide analysis) fan out to multiple shards in parallel and merge-sort by score before re-ranking — bounded by how many companies are in a single comparison request (typically single digits), so fan-out stays cheap.
- **Re-ranking** (cross-encoder, per the base doc's short-term roadmap) runs only on the merged top-k (e.g., top 50-100 candidates) after retrieval narrows the field — never on the full shard result sets, since cross-encoders are far more expensive per-item than the initial ANN search.

**Tradeoff:** pre-filtering trades a small amount of recall (a relevant chunk in a shard excluded by an overly strict filter is unreachable) for a very large latency and cost win. Mitigate by keeping filters advisory-widenable — if pre-filtered search returns too few results, fall back to a broader search rather than returning nothing.

## 5. Freshness vs. consistency

With continuous ingestion, there's a real question of how fresh the index needs to be relative to writes.

- **Eventual consistency for the vector store** is the right default: a newly ingested filing being searchable within seconds-to-minutes (not milliseconds) is acceptable — nobody expects to query a filing before the ingestion pipeline has finished processing it, and enforcing strong consistency here would mean synchronous writes across every replica of every affected shard, tanking ingestion throughput for no real user benefit.
- **Strong consistency for the metadata store's versioning** matters more — if a document is superseded by an amendment, queries should not non-deterministically return the stale version. Keep `superseded_by` pointers authoritative in the metadata store and have the RAG pipeline check version validity even if a stale chunk is still physically present in the vector index (compaction/deletion lags acceptably behind logical supersession).

**Tradeoff:** this two-speed consistency model (eventual for vectors, strong for version pointers) is more moving parts than a single consistency guarantee everywhere, but a single strong-consistency guarantee across a multi-billion-vector distributed index would be prohibitively slow to maintain, while a single eventual-consistency guarantee everywhere risks serving answers built on officially-superseded numbers — neither uniform choice is acceptable for financial data.

## 6. Cost shape at this scale

The dominant costs shift as scale grows:

| Scale | Dominant cost |
|---|---|
| Base doc (thousands of docs) | LLM generation calls |
| Millions of documents | Vector storage + embedding compute + metadata store partitioning overhead |

This is why the tiered-index and self-hosted-embedding tradeoffs above matter more here than at the base doc's scale — at millions of documents, storage and embedding costs compound continuously (every new filing adds permanent storage + one-time embedding cost), while LLM generation cost is per-query and doesn't grow with corpus size. Optimizing storage/embedding cost has a much bigger multiplier at this scale than optimizing generation cost does.

## Summary of Core Tradeoffs

| Decision | Chose | Traded away | Because |
|---|---|---|---|
| Kafka-buffered async ingestion | Absorbs bursty earnings-season load, replay-able | Seconds-to-minutes ingestion latency vs. synchronous | Nothing requires sub-second freshness; bursts would stall a synchronous pipeline |
| Self-hosted embedding fleet | Much lower marginal cost at billions of chunks | Infra ownership (GPU fleet, model drift) | Crossover point favors self-hosting well before millions-of-docs scale |
| Distributed vector store, sharded by time/ticker | Query load parallelizes, filtered queries hit few shards | Cross-shard queries (comparisons) cost more, added routing complexity | Common case (single-company/recent-quarter) dominates query volume |
| Tiered indexing (HNSW hot / IVF-PQ cold) | Massive memory savings on the long tail of old filings | Slightly lower recall on rarely-queried old data | Recent-quarter queries dominate; old-filing recall loss is low-impact |
| Object storage tiering by age | Large storage cost reduction | Higher latency retrieving very old raw filings | Rare access pattern, acceptable latency tradeoff |
| Partitioned distributed metadata store | Scales past a single Postgres box's ceiling | Cross-partition joins more expensive, operational complexity | Ticker-scoped queries (the common case) stay fast; comparisons are rarer |
| Metadata pre-filtering before ANN search | Large latency/cost win, narrows search to relevant shards | Small recall risk if filters are too strict | Mitigated with fallback-widening filters |
| Two-speed consistency (eventual vectors, strong version pointers) | Ingestion throughput stays high; version correctness stays strict | More operational complexity than one uniform model | Financial data can't tolerate stale/superseded numbers, but can tolerate seconds-old vector freshness |

The throughline: **the base doc's design (single-node vector store, synchronous ingestion) is correct for its stated scale and would be over-engineering at that scale if built this way from day one.** Every choice here is what changes specifically because the corpus stops fitting on one machine and stops arriving in discrete batches — sharding, tiering, and async ingestion are the direct consequences of "millions of documents, continuously," not general-purpose scaling advice.
