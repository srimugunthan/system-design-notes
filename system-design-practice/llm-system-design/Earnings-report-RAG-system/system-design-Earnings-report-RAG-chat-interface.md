# Chat Interface Design: Scaling for Latency & Throughput

## Context

This extends [system-design-Earnings-report-RAG.md](./system-design-Earnings-report-RAG.md), which defines the RAG pipeline as a single in-process call: `EarningsRAG.query(question) -> {answer, sources}`.

A chat UI changes the shape of the problem. Instead of one-shot request/response, we now need:
- **Multi-turn state** (conversation history, follow-up questions)
- **Streaming output** (tokens appear as generated, not after the full answer)
- **Many concurrent sessions**, each open for minutes, each bursty (idle while user reads, then a burst of retrieval + generation)

That combination — long-lived connections, streaming, and concurrency — is where naive designs fall over. This doc covers how to design the UI↔backend interface so it scales, and the tradeoffs at each decision point.

## Target SLOs (assume, then design against them)

| Metric | Target | Why it matters |
|---|---|---|
| Time to first token (TTFT) | < 800ms p50, < 2s p99 | Perceived responsiveness — the single biggest driver of "does this feel fast" in chat UIs |
| Inter-token latency | < 50ms p50 | Smooth streaming vs. stuttering |
| Full answer latency | < 6s p50, < 15s p99 | Retrieval + rerank + generation end-to-end |
| Throughput | 500+ concurrent sessions per region | Realistic team/org scale |
| Availability | 99.9% | Chat is often on the critical path of a workflow, not a toy |

Every design choice below is a tradeoff against one or more of these numbers — there is no single "correct" architecture, only the right one for which SLO you're protecting.

## High-Level Architecture

```
┌────────────┐      persistent conn        ┌─────────────────────┐
│  Chat UI   │ ───────────────────────────▶│   Chat Gateway (N)   │
│ (browser)  │◀─────────────────────────── │  stateless, autoscaled│
└────────────┘      SSE/WS stream          └──────────┬───────────┘
                                                        │
                                    ┌───────────────────┼───────────────────┐
                                    ▼                   ▼                   ▼
                          ┌──────────────┐   ┌──────────────────┐  ┌───────────────┐
                          │ Session Store │   │  Query Queue     │  │  Rate Limiter │
                          │ (Redis)       │   │  (backpressure)  │  │  (Redis/token │
                          └──────────────┘   └────────┬─────────┘  │   bucket)     │
                                                        ▼            └───────────────┘
                                            ┌───────────────────────┐
                                            │   RAG Pipeline Workers │
                                            │  (horizontally scaled) │
                                            │  Retriever → LLM Chain │
                                            └───────────┬────────────┘
                                                        ▼
                                   ┌─────────────────────────────────────┐
                                   │   Vector Store (sharded/replicated)  │
                                   │   LLM Inference (batched serving)    │
                                   │   Response Cache / Semantic Cache    │
                                   └─────────────────────────────────────┘
```

## 1. Transport: SSE vs WebSocket vs gRPC streaming

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **SSE** | HTTP/1.1+2 native, plain load balancers, auto-reconnect built into browser `EventSource`, simple to scale behind standard LBs | One-directional only (server→client); need a separate POST for client→server messages | **Default choice.** Chat is mostly server→client streaming; the client side is low-frequency (one message per turn), so a plain POST to send + SSE to receive is enough |
| **WebSocket** | Bidirectional, supports mid-stream cancel, typing indicators, lower per-message overhead | Stateful connection complicates load-balancing (needs sticky routing or a connection-aware fabric), harder to horizontally autoscale, proxies/CDNs handle it less gracefully | Use only if you need cancel-mid-generation or multiplexed low-latency signals beyond tokens |
| **gRPC streaming** | Efficient binary framing, strong typing | Browser support needs grpc-web translation layer, adds an extra hop/proxy | Better fit for service-to-service (gateway→worker), not for browser-facing edge |

**Tradeoff being made:** SSE trades bidirectional richness for operational simplicity at scale. At 500+ concurrent long-lived connections, "can a stateless LB round-robin this" matters more than shaving a few ms off signaling — so SSE for UI-facing, and keep WebSocket in your pocket only if the product needs cancel/typing-indicator features badly enough to pay the routing complexity.

## 2. Chat Gateway: stateless compute, external state

The gateway (the thing terminating the SSE connection) must be **stateless** — no session data in process memory. Why: stateless services scale horizontally by just adding replicas behind a load balancer; stateful ones require sticky sessions, which caps your ability to rebalance load and turns a single hot session into a hot instance.

- Session state (conversation history, active filters) → **Redis**, keyed by `session_id`, TTL'd (e.g., 30 min idle expiry).
- Gateway replicas are interchangeable; any replica can serve any session's next message by reading from Redis.

**Tradeoff:** every turn now pays a Redis round-trip (~1-2ms) to load/save history. That's the cost of horizontal scalability — accepted because 1-2ms is negligible against the multi-second RAG latency budget, while sticky sessions would create load imbalance and single points of failure under bursty traffic.

## 3. Backpressure: queue in front of the RAG workers

Retrieval + rerank + LLM generation is the expensive, slow part (seconds), while accepting an HTTP connection is cheap (ms). Without a queue, a burst of concurrent chat messages will oversubscribe LLM/vector-store capacity and latency degrades for everyone simultaneously (no isolation between requests).

Put a **bounded queue** (e.g., Redis Streams, SQS, or an in-memory work queue per worker pool) between the gateway and the RAG pipeline workers:

- Gateway enqueues `{session_id, rewritten_query, filters}` and opens the SSE stream immediately (so the UI shows a "thinking" state right away — protects perceived TTFT even if actual processing is queued).
- Workers pull from the queue, process, and push tokens back to the gateway (via pub/sub — e.g., Redis Pub/Sub channel per session) which forwards them onto the open SSE connection.

**Tradeoff:** this adds a hop (worker → pub/sub → gateway → client) instead of the worker writing directly to the client connection. The cost is a few ms of extra latency per token; the benefit is that gateways and workers scale independently — you can add RAG workers without touching connection-handling capacity, and vice versa. Direct-write-from-worker is simpler and fine at low scale, but couples your connection count to your inference capacity, which is exactly the coupling you want to break for throughput scaling.

**Also:** bound the queue and reject (HTTP 429 / SSE `error` event) once it's full, rather than letting it grow unboundedly — unbounded queues turn a throughput problem into a *latency* problem for every request currently waiting, including new ones. Fast, explicit failure beats slow degradation for every session.

## 4. Query rewriting and retrieval: where multi-turn cost hides

The Query Engine stage (rewriting a follow-up like "what about last quarter?" using conversation history) is a small LLM call of its own, sitting on the critical path before retrieval even starts.

- **Tradeoff — rewrite every turn vs. only when needed:** always rewriting adds a fixed LLM round-trip (~200-400ms) to every turn, even simple ones. A cheaper heuristic (rewrite only if the message is short / has pronouns / lacks a subject) cuts that cost for the common case but risks occasionally under-rewriting an ambiguous follow-up, degrading retrieval precision. For an earnings-report assistant where precision matters, bias toward always rewriting, but use a **small/fast model** (not the main answer-generation model) for this hop specifically to keep it cheap.

## 5. Scaling LLM inference: batching vs. per-request latency

This is the sharpest latency/throughput tradeoff in the whole system.

| Strategy | Throughput | Per-request latency | Notes |
|---|---|---|---|
| One request per inference call | Low (GPU underutilized) | Best possible TTFT | Fine at low QPS, wasteful at scale |
| Static batching (wait to fill a batch) | High | Worst TTFT (head-of-line blocking — fast requests wait for the batch to fill) | Avoid for interactive chat |
| **Continuous/dynamic batching** (vLLM, TGI, etc.) | High | Near-best TTFT — new requests join an in-flight batch at the next token step | Right choice for chat at scale |

**Tradeoff:** continuous batching requires a serving stack that supports it (vLLM, TensorRT-LLM, TGI) rather than a naive `model.generate()` loop — more operational surface area (GPU scheduling, KV-cache memory management) in exchange for getting both good p50 latency *and* high throughput instead of having to pick one.

Autoscale the inference layer on **queue depth / time-to-dequeue**, not just CPU/GPU utilization — GPU util can look "fine" while requests are still queuing because batches are full, which utilization metrics alone won't surface.

## 6. Caching: the highest-leverage latency win

| Cache layer | What it caches | Tradeoff |
|---|---|---|
| **Embedding cache** | Query → embedding vector | Cheap, low risk — identical text always embeds identically. Nearly free to add. |
| **Semantic response cache** | Similar questions (by embedding similarity, not exact match) → previously generated answers | Big latency win for common questions ("What was Apple's Q3 revenue?") but risks staleness — an earnings report doesn't change, but if it's amended/restated, cached answers go stale. Mitigate with cache invalidation tied to document ingestion (bust cache entries whose source doc changed) plus a TTL. |
| **Retrieval cache** | (query, filters) → top-k chunk IDs | Skips the vector search step on repeat/near-repeat queries. Safe as long as it's invalidated on new document ingestion. |

**Tradeoff, broadly:** caching trades a small risk of serving slightly stale answers for a large win on both latency (skip generation entirely on a cache hit) and throughput (fewer LLM calls = more headroom). For financial data where "don't make up information" is already a stated system requirement (see prompt template in the base doc), staleness risk must be bounded by tying cache invalidation to the document ingestion pipeline — never rely on TTL alone for anything tied to a specific quarter's filing.

## 7. Vector store scaling: sharding & read replicas

At the scale called out in the base doc's "Production Scale" section (~1M+ chunks), a single local FAISS/Chroma instance becomes both a latency and availability risk (no redundancy, single process bottleneck).

- **Read replicas** for the vector store: retrieval is read-heavy and tolerant of eventual consistency (new documents don't need to be searchable within milliseconds), so replicas scale read throughput cheaply.
- **Sharding by metadata** (e.g., by company or by year) keeps individual shard size bounded and lets metadata-filtered queries (`filter={'company': 'AAPL'}`) hit a single shard instead of scanning everything — turning a filtered query from "search all, then filter" into "search only the relevant shard."

**Tradeoff:** sharding adds routing complexity (the gateway/worker needs to know which shard(s) a query touches) and makes cross-shard queries (e.g., "compare AAPL and MSFT") more expensive since they now fan out to multiple shards and merge results. Accept this because the common case (single-company questions) gets faster, and the fan-out case is rarer and still bounded (a handful of shards, not all of them).

## 8. Degrading gracefully under load

Throughput ceilings will be hit eventually — the design question is what breaks first and how visibly.

- **Explicit rate limiting** (token bucket per user/session) with a clear SSE `error` event, rather than silent slowdowns — a fast, honest "you're being rate limited" beats a request that silently takes 30s.
- **Timeout budgets per stage** (retrieval, rerank, generation) so one slow stage doesn't consume the entire request budget silently — if retrieval overruns its budget, fail fast into a "couldn't find relevant context" response rather than proceeding into a generation call doomed to be slow anyway.
- **Partial results over total failure**: if generation is streaming and the connection or a downstream dependency fails mid-stream, the UI should keep whatever tokens arrived rather than discarding the whole answer — cheap to implement (client-side buffering) and meaningfully better UX under partial degradation.

## Summary of Core Tradeoffs

| Decision | Chose | Traded away | Because |
|---|---|---|---|
| SSE over WebSocket | Operational simplicity, easy horizontal scaling | Bidirectional richness (cancel, typing indicators) | Chat is server-heavy streaming; simplicity wins at scale |
| Stateless gateway + Redis session store | Horizontal scalability, no sticky routing | ~1-2ms per turn for state round-trip | Negligible vs. multi-second RAG latency |
| Queue between gateway and workers | Independent scaling of connection-handling vs. inference capacity | Extra hop, few ms latency | Decoupling lets you scale the bottleneck (inference) without over-provisioning the cheap part (connections) |
| Continuous batching for LLM serving | High throughput without sacrificing TTFT | Operational complexity (specialized serving stack) | Only way to get both good latency and good throughput simultaneously |
| Semantic/response caching | Big latency & throughput win on repeat questions | Staleness risk | Bounded by tying invalidation to document ingestion |
| Sharded vector store | Bounded shard size, faster filtered queries | Cross-shard queries cost more, added routing complexity | Common case (single-company) dominates; rare case (cross-company) stays acceptable |
| Explicit rate limiting & fail-fast timeouts | Predictable degradation, honest signal to users | Some requests rejected outright under load | Fast failure beats slow, silent degradation for everyone |

The throughline across every tradeoff: **push variability (queueing, batching, caching) into the backend where it can be managed centrally, and keep the client-facing contract (SSE stream of `token` / `sources` / `done` / `error` events) simple and stable** — so scaling decisions on the backend never require UI changes.
