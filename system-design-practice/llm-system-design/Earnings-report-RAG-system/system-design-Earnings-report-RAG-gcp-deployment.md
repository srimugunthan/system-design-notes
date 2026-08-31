# GCP Deployment: Global, Millions-of-Documents, High-Throughput, High-Availability

## Context

This maps the design in [system-design-Earnings-report-RAG.md](./system-design-Earnings-report-RAG.md), [system-design-Earnings-report-RAG-chat-interface.md](./system-design-Earnings-report-RAG-chat-interface.md), and [system-design-Earnings-report-RAG-backend-scaling.md](./system-design-Earnings-report-RAG-backend-scaling.md) onto concrete GCP services, deployed **multi-region and active-active**, sized for the backend-scaling doc's target (5-10M documents, 2-5B chunks) and the chat-interface doc's SLOs (sub-second TTFT, 1,000+ QPS, 99.9%+ availability).

Going global adds one more axis to every tradeoff already made in those docs: **which components can be single-region with global routing, and which must actually replicate data across regions.** That distinction drives most of the decisions below.

## Non-Functional Requirements Recap

| Requirement | Target | Source |
|---|---|---|
| Global latency | Users on any continent see TTFT < 1-2s | New — global users need geo-local compute, not just geo-local routing |
| Throughput | 1,000+ QPS sustained, spiky at earnings season | chat-interface doc |
| Scale | 5-10M documents, 2-5B chunks, ~30-60TB vectors | backend-scaling doc |
| Availability | 99.9%+, survive a full region outage | New — "high availability" implies regional failover, not just replica redundancy within a region |
| Consistency | Strong for version/supersession pointers, eventual for vector freshness | backend-scaling doc's two-speed model |

## GCP Service Mapping

| Component (from prior docs) | GCP Service | Why this one |
|---|---|---|
| Global entry point / LB | **Global External HTTPS Load Balancer** (Premium Tier, Anycast IP) | Single global IP, routes to nearest healthy regional backend, terminates TLS at Google's edge — this *is* the global-routing mechanism, not a CDN |
| Static assets + raw filings | **Cloud CDN** in front of **Cloud Storage** (multi-region bucket) | Matches the earlier CDN discussion: static UI bundle + immutable filing PDFs are the two things worth edge-caching |
| Ingestion queue (replaces Kafka) | **Pub/Sub** | Globally available by default, serverless, no partition/broker management — the backend-scaling doc's Kafka role, without the cluster ops |
| Ingestion workers | **Cloud Run** (or GKE Jobs for heavier batch runs) | Scales to zero between earnings-season bursts, scales out fast during them — matches the bursty ingestion profile directly |
| Chat Gateway (SSE termination) | **GKE (regional clusters, one per active region)** | Long-lived SSE connections need connection-level control (draining, custom autoscaling signals) that a stateless container platform like Cloud Run's per-request model handles less predictably at this connection count |
| Session store | **Memorystore for Redis** (regional, one per region) | Matches the chat-interface doc's Redis session store; kept region-local (see §3) rather than global |
| Query/task queue (gateway→RAG workers) | **Pub/Sub** (regional topics) or **Cloud Tasks** | Same backpressure role as the chat-interface doc's bounded queue |
| RAG worker pool | **GKE (regional, GPU/CPU node pools as needed)** | Matches "RAG Pipeline Workers" in both prior docs; autoscaled on Pub/Sub backlog, not just CPU |
| Vector store | **Vertex AI Vector Search** (managed) *or* self-hosted (Milvus/Weaviate) on GKE | See §4 — the sharpest tradeoff in this doc |
| Metadata store | **Cloud Spanner** (multi-region config) | Only GCP-native store that gives horizontal scale *and* strong consistency across regions in one system — directly serves the two-speed consistency model's "strong for version pointers" half |
| Embedding generation | **Vertex AI** (managed embedding endpoints) *or* self-hosted (TEI/vLLM) on GKE with GPU pools | Same managed-vs-self-hosted crossover argument as the backend-scaling doc, mapped onto GCP-specific services |
| LLM inference | **Vertex AI Model Garden endpoints** *or* self-hosted vLLM/TGI on GKE GPU pools | Same tradeoff, plus: Vertex AI endpoints give you managed continuous batching out of the box |
| Rate limiting / WAF / DDoS | **Cloud Armor** | Attaches directly to the Global LB; handles the chat-interface doc's rate-limiting requirement at the edge, before traffic reaches any region |
| Secrets (API keys, DB creds) | **Secret Manager** | Standard |
| CI/CD | **Cloud Build + Cloud Deploy**, images in **Artifact Registry** | Regional rollouts, canary/progressive delivery across GKE clusters |
| Observability | **Cloud Monitoring, Cloud Trace, Cloud Logging** | SLO dashboards keyed to the targets above; alerting on queue depth, TTFT p99, Spanner CPU, Vector Search QPS |
| Network isolation | **VPC Service Controls + Shared VPC** | Perimeter around Spanner/Vector Search/Storage so ingestion and query paths can't be reached outside the defined service boundary |

## Global Architecture

```
                                    ┌───────────────────────────────┐
                                    │   Global External HTTPS LB      │
                                    │   (Anycast IP, Cloud Armor,     │
                                    │    Cloud CDN for static/filings)│
                                    └───────────────┬─────────────────┘
                    ┌───────────────────────────────┼───────────────────────────────┐
                    ▼                                ▼                                ▼
        ┌───────────────────────┐      ┌───────────────────────┐      ┌───────────────────────┐
        │  us-central1            │      │  europe-west1           │      │  asia-southeast1        │
        │  ┌─────────────────┐    │      │  ┌─────────────────┐    │      │  ┌─────────────────┐    │
        │  │ GKE: Chat Gateway│    │      │  │ GKE: Chat Gateway│    │      │  │ GKE: Chat Gateway│    │
        │  └────────┬────────┘    │      │  └────────┬────────┘    │      │  └────────┬────────┘    │
        │  ┌────────▼────────┐    │      │  ┌────────▼────────┐    │      │  ┌────────▼────────┐    │
        │  │ Memorystore Redis│    │      │  │ Memorystore Redis│    │      │  │ Memorystore Redis│    │
        │  │ (session, region-│    │      │  │ (session, region-│    │      │  │ (session, region-│    │
        │  │  local)          │    │      │  │  local)          │    │      │  │  local)          │    │
        │  └─────────────────┘    │      │  └─────────────────┘    │      │  └─────────────────┘    │
        │  ┌─────────────────┐    │      │  ┌─────────────────┐    │      │  ┌─────────────────┐    │
        │  │ Pub/Sub: query   │    │      │  │ Pub/Sub: query   │    │      │  │ Pub/Sub: query   │    │
        │  │ queue (regional) │    │      │  │ queue (regional) │    │      │  │ queue (regional) │    │
        │  └────────┬────────┘    │      │  └────────┬────────┘    │      │  └────────┬────────┘    │
        │  ┌────────▼────────┐    │      │  ┌────────▼────────┐    │      │  ┌────────▼────────┐    │
        │  │ GKE: RAG Workers │    │      │  │ GKE: RAG Workers │    │      │  │ GKE: RAG Workers │    │
        │  └────────┬────────┘    │      │  └────────┬────────┘    │      │  └────────┬────────┘    │
        │  ┌────────▼────────┐    │      │  ┌────────▼────────┐    │      │  ┌────────▼────────┐    │
        │  │ Vertex AI Vector │    │      │  │ Vertex AI Vector │    │      │  │ Vertex AI Vector │    │
        │  │ Search (regional │    │      │  │ Search (regional │    │      │  │ Search (regional │    │
        │  │ read replica)    │    │      │  │ read replica)    │    │      │  │ read replica)    │    │
        │  │ + Vertex AI LLM  │    │      │  │ + Vertex AI LLM  │    │      │  │ + Vertex AI LLM  │    │
        │  └─────────────────┘    │      │  └─────────────────┘    │      │  └─────────────────┘    │
        └───────────┬─────────────┘      └───────────┬─────────────┘      └───────────┬─────────────┘
                    │                                │                                │
                    └────────────────────────────────┼────────────────────────────────┘
                                                       ▼
                            ┌────────────────────────────────────────────────┐
                            │        GLOBAL / MULTI-REGION SHARED LAYER        │
                            │  Cloud Spanner (multi-region: metadata, versions)│
                            │  Cloud Storage (multi-region: raw filings)       │
                            │  Pub/Sub (global topic: ingestion events)        │
                            └───────────────────┬──────────────────────────────┘
                                                 ▼
                            ┌────────────────────────────────────────────────┐
                            │   INGESTION: Cloud Run workers (autoscaled),     │
                            │   consume from Pub/Sub, write to Spanner +       │
                            │   Cloud Storage + push vector updates to each    │
                            │   region's Vector Search index                  │
                            └────────────────────────────────────────────────┘
```

## 1. Global entry & routing

The **Global External HTTPS Load Balancer** is the mechanism that makes "global" actually mean something here, distinct from the earlier CDN discussion: it uses a single Anycast IP and routes each user to the *nearest region with healthy backends*, using GCP's backbone rather than the public internet for the cross-region hop. This is what gets a user in Singapore routed to `asia-southeast1` automatically, with automatic failover to the next-nearest healthy region if that region's backends fail health checks.

- **Cloud CDN** sits in front of the static-asset and filings buckets (per the earlier CDN discussion) — attached directly to this same LB, no separate infrastructure.
- **Cloud Armor** attaches here too: rate limiting and WAF rules run *before* traffic reaches any region, so an abusive client gets rejected at the edge instead of consuming regional GKE/Vector Search capacity.

**Tradeoff:** Premium Tier global LB costs more than per-region regional LBs, but regional LBs would require the client (or DNS) to pick a region — pushing geo-routing logic into the client, which is both worse (no health-aware failover) and more complex than letting Google's network do it.

## 2. Chat Gateway: GKE over Cloud Run, specifically for this workload

Cloud Run is the simpler default for stateless HTTP services on GCP, and would be the right call for the ingestion workers (§ mapping table). For the **chat gateway** specifically, GKE is worth the extra operational surface because:

- SSE connections here are long-lived (minutes) and the chat-interface doc's design depends on precise control of connection draining during deploys (finish in-flight streams before terminating a pod) — GKE's pod lifecycle hooks give this directly; Cloud Run's request-based model is a closer fit for short request/response cycles.
- Autoscaling needs to key off **Pub/Sub queue depth**, not just request count or CPU (same point the chat-interface doc makes about inference autoscaling) — GKE's HPA supports custom Cloud Monitoring metrics natively.

**Tradeoff:** GKE means cluster upgrades, node pool management, and more YAML versus Cloud Run's zero-ops model. Accepted here specifically because connection-level control matters for SSE at this scale; if the product later drops to simple request/response (no streaming), Cloud Run would be the better default.

## 3. Session store: region-local, not global

Memorystore for Redis is deployed **per region**, holding only that region's active sessions — not replicated globally.

**Tradeoff:** if a user's request fails over to a different region mid-session (rare — only on a regional outage), their conversation history is lost from that region's Redis and a fresh session starts. Accepted because: (a) full cross-region session replication would add write latency to *every* chat turn to keep replicas in sync, directly hurting the TTFT SLO, for a benefit (surviving mid-conversation regional failover) that affects a small fraction of sessions during a rare event; (b) the conversation history is disposable state — worst case is "start the chat over," not data loss of anything durable. This is the same reasoning as the chat-interface doc's "Redis round-trip is cheap, sticky global consistency is not worth it," just applied across regions instead of across gateway replicas.

## 4. Vector store: the central regional-replication decision

This is where "global" and "millions of documents" collide hardest. Two real options:

| Approach | How it works | Tradeoff |
|---|---|---|
| **Vertex AI Vector Search, one index per region, fed by the same ingestion pipeline** | Ingestion (in one canonical region) writes to Spanner + Cloud Storage, then pushes index updates to each region's Vector Search deployment | Each region serves queries against a **local, low-latency** index — no cross-region hop on the query path, which is the whole point of going global. Cost: multiple copies of the same ~30-60TB index (one per active region) and near-real-time (not instant) propagation of new documents to non-primary regions |
| **Single global Vector Search deployment, regions query it remotely** | One canonical index, all regions' RAG workers query it over GCP's backbone | Simpler — one index, no propagation lag, no extra storage cost | Every non-local region pays a cross-region round-trip on every retrieval call, directly hurting TTFT for exactly the users global deployment was supposed to help |

**Chosen: per-region replicated indexes.** The whole reason to deploy globally is to keep compute (and now the index) close to users; a single global index would silently undo that benefit for every region except the primary one. The cost — running the index 3x and tolerating a propagation lag (new filings visible in secondary regions minutes after the primary, consistent with the backend-scaling doc's "eventual consistency for vectors is fine" position) — is the right trade given the TTFT SLO is the whole point of this deployment.

**Managed vs. self-hosted, mapped from the backend-scaling doc's tradeoff:** Vertex AI Vector Search is the GCP-native managed option (no cluster ops, built-in tiered indexing under the hood) and is the default recommendation here specifically *because* running self-hosted Milvus/Weaviate replicated across three regions multiplies the operational burden the backend-scaling doc already flagged as significant at single-region scale. Self-hosting on GKE remains the right call only if there's a specific need Vertex AI Vector Search doesn't cover (e.g., a specific ANN algorithm or hybrid-search feature) — at that point, replicate the same way: one deployment per region, fed by the same canonical ingestion pipeline.

## 5. Metadata store: Cloud Spanner, multi-region config

Spanner is the one component that should genuinely be **one logical global database**, not per-region copies — because the backend-scaling doc's two-speed consistency model requires the `superseded_by` version pointers to be **strongly consistent everywhere**. A per-region copy of metadata risks region A serving a superseded filing version after region B has already recorded its amendment.

- Spanner's multi-region configuration (e.g., `nam-eur-asia1`) replicates synchronously with external consistency guarantees, giving every region a strongly consistent read of version state without building custom replication logic.
- Spanner also directly satisfies the backend-scaling doc's "partition, don't scale vertically" requirement — it shards automatically, removing the need to hand-manage partitions the way a Citus/Postgres setup would.

**Tradeoff:** Spanner's multi-region writes have higher latency than a single-region database (consensus across regions costs tens of ms per write) and Spanner is priced at a premium versus Cloud SQL/AlloyDB. Accepted because metadata writes (new filing recorded, version superseded) are low-frequency relative to reads and relative to the retrieval/generation path — paying tens of ms on an infrequent write is a much better trade than risking a stale-version read on every query, which is exactly the failure mode the backend-scaling doc calls out as unacceptable for financial data.

**Cheaper alternative if global strong consistency isn't actually required:** AlloyDB (single-region primary + read replicas) is meaningfully cheaper and simpler, at the cost of a single point of write failure and cross-region reads being either stale (replica lag) or slow (reading the primary remotely). Worth reconsidering if the product's actual consistency needs turn out to be looser than "never serve a stale version."

## 6. Ingestion pipeline: same design, GCP-native pieces

The backend-scaling doc's Kafka-buffered, worker-pool ingestion pipeline maps directly:

- **Pub/Sub** replaces Kafka as the durable, replay-able ingestion buffer — no partition/broker management, and it's globally available by default (a single topic can be published to and subscribed from any region), simplifying the "one canonical ingestion region" model in §4.
- **Cloud Run** (scale-to-zero, fast burst-scaling) replaces the generic "worker pool" for the load→parse→chunk→embed→version→store steps — a good fit given ingestion is bursty (earnings season) and otherwise idle, exactly the profile Cloud Run is priced and built for.
- Embedding generation within these workers uses the same managed-vs-self-hosted tradeoff as the backend-scaling doc (Vertex AI embedding endpoints vs. self-hosted TEI/vLLM on GKE GPU pools) — the crossover point toward self-hosting still applies once ingestion volume is continuously high, independent of which cloud it runs on.

## 7. High availability specifics

- **Regional failover**: the Global LB's health checks pull an unhealthy region out of rotation automatically; combined with per-region Vector Search replicas and Spanner's multi-region durability, a full region outage degrades capacity but not correctness — surviving users are rerouted to the next-nearest healthy region.
- **Zonal redundancy within a region**: GKE clusters and Vector Search deployments span multiple zones per region by default — a single zone failure doesn't take out a region.
- **Spanner's built-in HA**: multi-region Spanner survives a full region loss with no manual failover (it's designed for this), which is why it was chosen over a self-managed distributed SQL cluster despite the cost premium.
- **Progressive delivery**: Cloud Deploy rolls changes through regions sequentially (canary in one region, verify SLOs, then promote), rather than a global simultaneous deploy — so a bad release is caught in one region's blast radius before it reaches all three.

## Summary of Core Tradeoffs

| Decision | Chose | Traded away | Because |
|---|---|---|---|
| Global External HTTPS LB (Premium Tier) | Automatic nearest-region + health-aware routing | Higher cost than regional LBs | Client-side/DNS geo-routing is worse and more complex |
| GKE for chat gateway (not Cloud Run) | Precise SSE connection lifecycle control, custom-metric autoscaling | More operational surface than a serverless platform | Long-lived streaming connections need that control; ingestion workers stay on Cloud Run where it's the better fit |
| Region-local Redis sessions (no global replication) | Low-latency session read/write on every chat turn | Session loss on the rare cross-region failover | Disposable state; replication cost would hit TTFT on every turn to protect a rare-event edge case |
| Per-region replicated Vector Search indexes | Local, low-latency retrieval in every region | 3x index storage cost, minutes of cross-region propagation lag | The entire point of going global is keeping compute (and now the index) close to users |
| Managed Vertex AI Vector Search over self-hosted, multi-region | Avoids multiplying self-hosted ops burden across 3 regions | Less control over specific ANN internals | Self-hosting's ops cost (flagged even at single-region scale in the backend-scaling doc) multiplies with each additional region |
| Cloud Spanner, multi-region, for metadata | Strong consistency for version pointers globally | Write latency premium, higher cost than AlloyDB/Cloud SQL | Financial data can't tolerate a stale/superseded version being served anywhere; version writes are infrequent enough to absorb the cost |
| Pub/Sub over self-managed Kafka | No broker/partition ops, global availability by default | Less fine-grained control than a hand-tuned Kafka cluster | Same buffering/replay role, with GCP's ops burden instead of the team's |

The throughline: **going global doesn't change most of the backend-scaling doc's decisions — it forces a new question on top of each one: replicate the data, or replicate the compute and centralize the data?** The answer split cleanly along consistency requirements — Spanner (metadata/versions) stayed centralized-but-globally-consistent because correctness demands it; Vector Search and Redis went per-region because latency demands it and their consistency requirements are already loose (eventual vector freshness, disposable session state) per the backend-scaling doc's own two-speed model.
