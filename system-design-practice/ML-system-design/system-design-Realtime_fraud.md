# System Design: Real-Time Fraud Alert & Case Management Platform

## 1. Problem Statement

Design a platform that:
- Ingests transaction/event streams (card, wire, ACH, UPI, P2P) in real time
- Scores/flags suspicious activity (fed by an upstream ML scoring service or rules engine)
- Raises **alerts** and routes them to fraud analysts
- Lets analysts **investigate, annotate, escalate, and disposition** alerts as **cases**
- Pushes live updates to analyst dashboards (new alert, SLA breach, case reassignment)
- Meets bank-grade requirements: auditability, low false-negative tolerance, regulatory reporting (SAR/STR), and strict latency SLAs
- ## System Design Question

**Design a real-time fraud alert and case management platform for a bank's fraud operations team**, where transaction events from multiple payment channels (card, wire, UPI/ACH) stream in continuously and must be scored, alerted, and surfaced to fraud analysts with sub-second latency, while supporting millions of daily transactions and thousands of concurrent analyst sessions.

Your design should address:

**Non-functional requirements / building for scale**
- How would you protect the ingestion endpoints (auth, rate limiting, DDoS/abuse protection) that receive transaction events from many upstream channels?
- How would you shard and index the transaction/case database to support both high-throughput writes and fast analyst-facing queries (e.g., "show all cases for this customer in the last 90 days")? Would you use a DB proxy layer, and why?
- How would you load balance across your scoring/inference services and API tier, and what happens on a regional failover?

**Domain services design**
- How would you decompose this into domain services (e.g., ingestion, fraud-scoring, case management, notification, audit)? What owns the transaction data vs. the case data vs. the analyst-facing view?
- How do these services communicate — synchronous APIs vs. async events — and how do you avoid tight coupling between the scoring engine and case management?

**Messaging, caching, and real-time delivery**
- What messaging backbone would you use to move transaction events from ingestion to scoring to case creation (e.g., Kafka), and how do you handle backpressure or a slow consumer (case management) without dropping fraud signals?
- Where would you introduce caching (customer risk profiles, model features, session state), and how do you keep it consistent with the source of truth?
- How would analysts receive live updates on new fraud cases — polling, WebSockets, or server-sent events — and what are the tradeoffs given thousands of concurrent analyst dashboards?

**Latency, throughput, networks/protocols**
- What's your end-to-end latency budget from "transaction observed" to "alert visible to analyst," and where does each hop (network, queue, scoring, DB write, push to UI) eat into that budget?
- Would you use gRPC, REST, or a mix internally vs. externally, and why?
- How do you handle throughput spikes (e.g., festive season transaction surges) without breaching your latency SLA?

This is a good one because it forces trade-offs across every layer you listed — it's not just "design Kafka" or "design a cache," but a coherent system where scoring latency, messaging durability, DB sharding strategy, and real-time UI delivery all pull against each other, similar to what you'd face designing something like ContagionMapper or Shield-Fin at production scale.

### Functional Requirements
1. Real-time alert ingestion from scoring engines (1 or many)
2. Case creation, assignment, workflow states (New → In Review → Escalated → Closed/SAR-filed)
3. Analyst workbench: search, filter, bulk actions, comments, attachments
4. Real-time push of new/updated alerts to connected analysts (queue view)
5. Full audit trail (who viewed/changed what, when) — regulatory requirement
6. Reporting/analytics (SLA adherence, analyst throughput, false-positive rate)
7. Integration with case disposition feeding back into the ML model (label feedback loop)

### Non-Functional Requirements
| Attribute | Target |
|---|---|
| Alert ingestion → analyst screen latency (p99) | < 2s |
| Alert ingestion throughput | 5,000–20,000 events/sec peak (card-network scale) |
| Case DB read latency (workbench queries) | < 200ms p95 |
| Availability | 99.95% (fraud ops is 24x7) |
| Durability | Zero alert loss (regulatory) |
| Audit log retention | 7+ years (compliance) |

---

## 2. High-Level Architecture

```
                                   ┌─────────────────────────┐
                                   │   Upstream Sources       │
                                   │  Card Auth / Wire / UPI  │
                                   │  ML Scoring Engine       │
                                   └────────────┬─────────────┘
                                                │  gRPC/HTTPS (mTLS)
                                                ▼
                              ┌──────────────────────────────────┐
                              │      API Gateway / Ingress        │
                              │  (mTLS term., authN, rate-limit)  │
                              └────────────────┬───────────────────┘
                                                │
                                                ▼
                              ┌──────────────────────────────────┐
                              │     Alert Ingestion Service        │
                              │  (validate, dedup, enrich)         │
                              └───────┬──────────────┬─────────────┘
                                      │              │
                                      ▼              ▼
                         ┌───────────────────┐  ┌─────────────────────┐
                         │  Kafka: alerts.raw │  │  Kafka: alerts.dlq   │
                         └────────┬──────────┘  └──────────────────────┘
                                  │
              ┌───────────────────┼─────────────────────┐
              ▼                   ▼                      ▼
   ┌───────────────────┐  ┌───────────────┐   ┌───────────────────────┐
   │ Case Orchestrator  │  │ Notification  │   │ Feature/Enrichment     │
   │ (assignment rules, │  │ Service (push  │   │ Consumer (customer     │
   │  SLA timers, state │  │ via WS/SSE +   │   │ 360, device, geo)      │
   │  machine)          │  │ email/SMS)     │   └────────────┬────────────┘
   └─────────┬──────────┘  └───────┬───────┘                 │
             │                     │                          │
             ▼                     ▼                          ▼
   ┌────────────────────────────────────────────────────────────────┐
   │                    Case Management Service                      │
   │        (CRUD cases, comments, attachments, disposition)         │
   └───────────────┬─────────────────────────────┬────────────────────┘
                   │                              │
                   ▼                              ▼
      ┌────────────────────────┐      ┌──────────────────────────┐
      │  DB Proxy (PgBouncer/  │      │   Cache Layer (Redis)     │
      │  ProxySQL) → Sharded   │      │   hot alerts, session,     │
      │  Postgres (cases,      │      │   analyst queue state       │
      │  alerts, audit)        │      └──────────────────────────┘
      └────────────────────────┘
                   │
                   ▼
      ┌────────────────────────┐
      │  Audit/Event Store      │
      │  (append-only, WORM)     │
      └────────────────────────┘

                                   ┌───────────────────────────┐
                                   │  Analyst Web App            │
                                   │  (WebSocket for live push,   │
                                   │   REST for CRUD/search)      │
                                   └───────────────────────────┘
```

---

## 3. Endpoint Protection

The platform has two very different endpoint surfaces, each needing different controls.

### 3.1 Machine-to-machine ingress (scoring engines, core banking, card network)
- **mTLS everywhere**: every upstream producer authenticates with a client cert issued by an internal CA; the gateway terminates/validates it. No shared static API keys for high-trust internal feeds.
- **API Gateway (Kong/Envoy/AWS API GW)** in front of the ingestion service does:
  - AuthN (mTLS or OAuth2 client-credentials for lower-trust feeds)
  - Coarse-grained authZ (which source is allowed to publish which event types)
  - Rate limiting per source (token bucket) to protect against a misbehaving upstream flooding the pipeline
  - Payload schema validation (reject malformed events before they hit Kafka)
  - Request size caps, JSON-bomb/zip-bomb protection
- **WAF** in front of the gateway for the public-facing analyst app (OWASP Top 10 rules, bot protection, geo-fencing to bank's operating regions).
- **mTLS + service mesh (Istio/Linkerd)** internally between microservices — no service trusts another based on network location alone (zero-trust internal segmentation), each service has its own workload identity (SPIFFE/SPIRE).

### 3.2 Human endpoint (analyst workbench)
- SSO via bank's IdP (SAML/OIDC), enforced MFA (fraud ops is a Tier-0 sensitive function — insider-threat surface).
- Session tokens short-lived (15 min) with silent refresh; WebSocket connections re-authenticate on reconnect.
- **Device posture checks** (managed device only, no BYOD) since analysts see full PII/PAN data.
- Row/field-level authorization: analysts only see cases in their assigned queue/region unless elevated (least privilege); PAN/account numbers masked by default, unmask requires a step-up action that's itself audited.
- CSP, strict cookie flags (HttpOnly, Secure, SameSite=strict), CSRF tokens for state-changing REST calls.
- All endpoint activity (view, unmask, export, disposition) logged to the immutable audit store — this doubles as insider-threat detection surface.

---

## 4. DB Proxy / Sharding / Indexing

### 4.1 Why a DB proxy
Direct app→DB connections don't scale with many stateless service instances (connection storms) and make failover/read-write splitting hard to manage app-side.
- **PgBouncer** (transaction pooling mode) sits in front of Postgres: absorbs bursty connection counts from horizontally-scaled case/alert services into a small stable pool against the DB.
- Proxy also handles **read/write splitting**: writes (new case, disposition) → primary; workbench search/list queries → read replicas.
- Enables **online failover**: proxy re-points to new primary during a promotion without every service redeploying connection strings.

### 4.2 Sharding strategy
Fraud/case data grows unbounded (years of transaction-linked alerts, regulatory retention). Single-node Postgres won't hold it at bank scale.
- **Shard key: `customer_id` (hashed)** — most workbench queries and almost all fraud investigation is customer-centric ("show me all alerts/cases for this customer"), so this keeps an investigation's data co-located on one shard (avoids fan-out joins across shards for the hottest query pattern).
- Alternative considered: shard by `region`/`business_unit` — rejected as primary key because customer activity isn't region-bound (cross-border wires), but region is kept as a **secondary index** for compliance-driven queries (e.g., "all EU cases this quarter").
- Use **Citus (Postgres extension)** or **Vitess-style** sharding rather than hand-rolled sharding, to keep transactional guarantees and SQL surface.
- **Audit/event log is NOT sharded the same way** — it's append-only and partitioned by **time (monthly partitions)** since access pattern there is "all events for time range X" for regulators, not customer lookups. Old partitions age out to cold/WORM storage (S3 Glacier-class) after N months but remain queryable via a separate archive path.

### 4.3 Indexing
- `alerts`: composite index on `(status, assigned_analyst_id, created_at)` — this is the literal query the analyst queue view runs every few seconds.
- `alerts`: index on `(customer_id, created_at DESC)` for 360-degree customer view.
- `cases`: index on `(sla_due_at)` where `status != 'closed'` — **partial index**, since the SLA-breach scanner only cares about open cases; keeps the index small and fast as closed cases pile up over years.
- GIN index on a `tags`/`typology` JSONB column for flexible fraud-typology search (e.g., "account takeover", "mule account") without a rigid schema per typology.
- Avoid over-indexing write-heavy `alerts.raw` staging table — that table is a thin landing zone, not queried directly by users.
- Full-text search on case notes offloaded to **Elasticsearch/OpenSearch** (CDC from Postgres via Debezium) rather than Postgres `tsvector`, since analysts need fuzzy/typo-tolerant search across free-text investigation notes at scale.

---

## 5. Load Balancing

Layered LB, matching each traffic pattern:

1. **Edge / DNS-level (GSLB)**: routes to nearest healthy region (multi-region for DR — fraud ops cannot go dark).
2. **L7 Load Balancer (ALB/Envoy)** in front of API Gateway: TLS termination, path-based routing (`/alerts/*` vs `/cases/*` vs `/ws/*` to different backend pools), health-check based ejection.
3. **WebSocket-aware LB**: sticky routing (consistent hashing on `analyst_session_id`) so a reconnect lands back on a gateway node that still has warm subscription state where possible — but design the notification service to be stateless-recoverable (see §8) so a miss isn't fatal, just costs one resubscribe round-trip.
4. **Service mesh L7 LB (client-side, via sidecar)** between internal microservices: round-robin/least-outstanding-requests with circuit breaking (Envoy) — prevents one slow Case Service pod from being hammered while others idle.
5. **Kafka partition-level "load balancing"**: not a traditional LB, but partitioning key (`customer_id`) distributes ingestion load across brokers/consumers while preserving per-customer ordering.
6. **DB read replicas behind the proxy**: PgBouncer/proxy layer round-robins read-only workbench queries across replicas, writes always to primary.

Health checks are **deep** (not just TCP): the ingestion service's health check verifies it can actually reach Kafka, not just that the process is up — avoids routing traffic to a pod that's "alive but useless."

---

## 6. Messaging

Kafka (or equivalent, e.g., MSK/Confluent) is the backbone connecting ingestion → scoring/enrichment → case orchestration → notification, decoupling producers/consumers and giving durability.

### Topic design
| Topic | Key | Purpose | Retention |
|---|---|---|---|
| `alerts.raw` | `customer_id` | Raw incoming alerts from scoring engines | 7 days (replay buffer) |
| `alerts.enriched` | `customer_id` | After enrichment (device/geo/customer-360 join) | 7 days |
| `cases.events` | `case_id` | State transitions (assigned, escalated, closed) — event-sourced case history | Long/compacted |
| `notifications.out` | `analyst_id` | Fan-out target for push service | Short (hours) |
| `alerts.dlq` | — | Failed validation/processing, for manual triage | Long |
| `audit.events` | `entity_id` | Every read/write/view for compliance | 7+ years (tiered storage) |

### Why Kafka specifically here
- **Ordering per customer** matters (an "account takeover" alert followed by a "large wire" alert on the same customer must be processed in order) → partition key = `customer_id`.
- **Replayability**: if the case-orchestration consumer has a bug and needs redeploy, we don't lose alerts — replay from offset. This is a hard regulatory requirement (cannot silently drop a fraud alert).
- **Backpressure isolation**: if the notification/push path is slow, it doesn't block the case-creation consumer — separate consumer groups.
- **`cases.events` as event-sourced log**: the case's current state is a projection over its event stream — gives us a free, tamper-evident audit trail for "how did this case get to Closed" for regulators, and lets us rebuild the case-state read model if it ever gets corrupted.

### Delivery semantics
- Producers: `acks=all`, idempotent producer enabled → no duplicate/lost alerts on retry.
- Consumers: process with **at-least-once** + idempotent upserts in Postgres (keyed on `alert_id`) → effectively exactly-once for case creation, without the complexity of transactional Kafka-DB writes everywhere.
- DLQ + alerting on DLQ depth — a growing DLQ on a fraud pipeline is itself a P1 incident.

---

## 7. Caching

Caching here is about **shaving latency off the hot analyst path** and **reducing DB load from repetitive reads**, never about being the source of truth for anything regulatory.

| Cache | What | TTL / invalidation |
|---|---|---|
| Redis: analyst queue snapshot | Precomputed "my open alerts sorted by priority" per analyst | Invalidated on new alert assignment / disposition event (via `cases.events` consumer updating cache) — not a blind TTL, since staleness here directly means an analyst misses something |
| Redis: customer-360 enrichment | Recent device/geo/risk-score lookups joined onto alerts | TTL ~5 min; enrichment data changes slowly relative to alert velocity |
| Redis: session/auth | Analyst session tokens, WS subscription registry | TTL = session length |
| CDN edge cache | Static workbench UI assets | Standard asset caching, cache-busted on deploy |
| **Never cached** | Case disposition status used for regulatory reporting, PAN/account data, audit trail | Always read from primary source of truth |

- **Cache-aside pattern** for customer-360: service checks Redis, falls back to DB/feature-store on miss, populates cache.
- **Write-through invalidation, not TTL-only, for the queue view** — because a stale "you have 3 alerts" when there are actually 5 is an operational fraud-miss risk, not just a UX nit. TTL alone is unacceptable for this specific cache; event-driven invalidation is mandatory.
- Guard against **cache stampede** on a hot customer (e.g., viral fraud ring investigation touching one entity from many analysts) with request coalescing / short-lived locks.

---

## 8. Polling vs Streaming vs WebSockets

This is the crux of "real-time alerting" and deserves an explicit decision, not a default.

| Option | Fit here | Why / why not |
|---|---|---|
| **Short-polling** (analyst UI hits `/alerts?since=` every N sec) | Fallback only | Simple, but 2–5s inherent latency and wasteful at scale (thousands of analysts × requests/sec against DB/cache even when nothing changed). Acceptable as a **degraded-mode fallback** if WS is unavailable. |
| **Long-polling** | Not chosen | Better than short-polling but still holds connections/threads open inefficiently compared to WS; no real advantage over WS given modern browser support. |
| **SSE (Server-Sent Events)** | Used for **one-way broadcast channels** (e.g., "system-wide SLA breach banner", "queue-level counts") | Simpler than WS, auto-reconnect built into the browser API, but one-directional — fine for notifications the analyst doesn't need to respond to over the same channel. |
| **WebSockets** | **Primary channel for the live alert queue** | Bidirectional (analyst can ack/claim an alert instantly over the same channel), low per-message overhead, true push — meets the <2s p99 requirement. Used for: new alert pushed to queue, case reassignment, SLA countdown ticks, presence ("analyst X is viewing this case" to prevent duplicate work). |
| **gRPC streaming** | Used **internally**, service-to-service (e.g., enrichment service streaming features to the scoring engine), not to the browser | Efficient binary framing, but browser support for gRPC-Web adds complexity not worth it for the analyst UI when WS already fits. |

**Reconnection design**: on WS reconnect, client sends `last_seen_event_id`; server replays any missed `cases.events`/`alerts.enriched` messages since then from Kafka/Redis before resuming live stream — this is what makes the WS layer safe to be non-sticky at the LB (§5) without losing alerts on a reconnect blip.

**Why not push everything through Kafka to the browser directly**: browsers can't be Kafka consumers safely (auth, protocol mismatch) — the **Notification Service acts as the bridge**: consumes `notifications.out`, fans out to the specific analyst's WS connection(s), tracks connection registry in Redis (`analyst_id → gateway node`) for routing across horizontally-scaled WS gateway pods.

---

## 9. Latency & Throughput

### Latency budget (target: ingest → analyst screen, p99 < 2s)

```
Upstream event received at gateway .......... 0ms
mTLS + gateway validation .................... +20ms   (cumulative: 20ms)
Ingestion service validate/dedup ............. +30ms   (50ms)
Kafka produce (acks=all) ..................... +20ms   (70ms)
Enrichment consumer (customer-360 lookup,
  cache-aside hit ~90% of the time) ........... +80ms   (150ms)
Case orchestrator (assignment rule eval,
  DB upsert via proxy) ......................... +150ms  (300ms)
Kafka produce to notifications.out ........... +20ms   (320ms)
Notification service fan-out to WS gateway ... +50ms   (370ms)
WS push to browser + render .................. +100ms  (470ms)
------------------------------------------------------------
Budget consumed: ~470ms typical, leaving ~1.5s headroom for:
  - GC pauses, network jitter, replica lag on read paths
  - Retry-on-transient-failure paths (e.g., a single Kafka retry)
```

Each hop above is instrumented (OpenTelemetry trace spans) so p50/p95/p99 per hop is visible — critical because "which stage ate the budget" is the actual operational question during an SLA-breach investigation, not just the end-to-end number.

### Throughput
- Target: 5,000–20,000 alert-worthy events/sec at peak (this is *post-scoring* alert volume, not raw transaction volume, which is far higher upstream and out of this system's scope).
- Kafka partition count sized so peak throughput per partition stays well under single-partition ceiling (~10s of MB/s) — e.g., 64 partitions on `alerts.raw` gives headroom and parallelism for consumer groups.
- Case Orchestrator and Enrichment consumers scaled horizontally (consumer group members ≤ partition count) — this is the main throughput lever, not vertical scaling.
- DB write path is the usual bottleneck at this throughput; mitigated by: batching upserts where safe, the PgBouncer pool absorbing connection overhead, and sharding (§4) spreading write load across shards by `customer_id` hash.
- Backpressure: if the Case Orchestrator falls behind, Kafka absorbs the buffer (consumer lag grows, but nothing is dropped) — lag is alerted on with a tight threshold given the "no lost alert" requirement.

---

## 10. Networks & Protocols

| Layer | Choice | Rationale |
|---|---|---|
| External ingress (upstream scoring engines → gateway) | HTTPS/gRPC over mTLS, TLS 1.3 | Regulated data in transit; mTLS gives mutual identity, not just encryption |
| Analyst browser ↔ backend (CRUD/search) | HTTPS REST (or GraphQL for flexible workbench queries), TLS 1.3 | Standard, cacheable, works with existing WAF/CDN tooling |
| Analyst browser ↔ backend (live push) | WSS (WebSocket over TLS) | See §8 |
| Service ↔ service (internal) | gRPC over mTLS (service mesh-issued certs) | Low-latency binary protocol + built-in mesh-level mutual auth, avoids re-solving authn per service |
| Service ↔ Kafka | TLS + SASL (SCRAM or mTLS) | Broker-level authn/authz per topic (e.g., only Case Orchestrator can consume `alerts.enriched`) |
| Service ↔ DB proxy | TLS, network-policy restricted (only case/alert services can reach the proxy, not the whole VPC) | Defense in depth even inside the perimeter |
| Cross-region replication (DR) | Private backbone / VPC peering, encrypted | Multi-region case data replication for failover, not exposed to public internet |
| Network segmentation | Separate VPC/subnet tiers: DMZ (gateway/WAF) → app tier → data tier, security groups deny-by-default | Standard defense-in-depth; fraud-ops data is a prime insider/external target |

**Protocol choice notes**:
- REST over GraphQL for most CRUD because case/alert schemas are well-defined and REST's caching semantics (ETags, CDN-friendliness for read-heavy list endpoints) matter more here than GraphQL's flexible-query benefit; GraphQL considered only if the workbench needs highly variable per-analyst dashboard composition.
- gRPC internally over REST for service-to-service because of lower serialization overhead (protobuf) at the throughput levels in §9, and native streaming support used by the enrichment pipeline.
- No raw TCP/UDP exposed anywhere externally — everything above HTTPS/WSS/gRPC-over-TLS at the edge.

---

## 11. Data Model (core entities, simplified)

```
alerts
  id (PK), customer_id, source_system, raw_score, typology,
  status [new|assigned|in_review|escalated|closed],
  created_at, assigned_analyst_id, case_id (FK, nullable until triaged)

cases
  id (PK), primary_customer_id, status, priority, sla_due_at,
  opened_at, closed_at, disposition [confirmed_fraud|false_positive|sar_filed],
  assigned_analyst_id, region

case_events (event-sourced, append-only)
  id, case_id, event_type, actor_id, payload (JSONB), occurred_at

audit_log (WORM / append-only, time-partitioned)
  id, entity_type, entity_id, actor_id, action, before, after, occurred_at
```

`alerts.case_id` is nullable because many alerts land un-triaged before an analyst (or an auto-triage rule) links them into a case — a case can aggregate multiple related alerts on the same customer/typology.

---

## 12. Failure Modes & Resilience Notes

- **Kafka broker loss**: replication factor ≥ 3, min.insync.replicas=2 — tolerates a broker failure with zero data loss.
- **DB primary failure**: automated failover (Patroni or cloud-managed) promotes a replica; PgBouncer reconnects; brief write unavailability window is acceptable, alert loss is not (Kafka buffers upstream during this window).
- **WS gateway pod crash**: analyst client auto-reconnects, replays missed events via `last_seen_event_id` (§8) — no alert is silently missed even if the exact delivery path was mid-flight.
- **Region outage**: active-passive (or active-active for read paths) DR; case data replicated cross-region; RPO target near-zero for `cases`/`audit_log`, RTO measured in minutes given regulatory expectation of continuous fraud-ops coverage.
- **Poison-pill message**: DLQ (§6) isolates it from blocking the partition; on-call gets paged on DLQ growth, not silent drop.

---

## 13. Summary of Key Design Decisions

1. **Kafka as the spine** decouples ingestion, enrichment, case logic, and notification — gives replay, ordering-per-customer, and backpressure isolation.
2. **WebSockets, not polling**, for the analyst queue — polling can't hit sub-2s p99 at this analyst-count scale without wasteful over-querying.
3. **Shard by `customer_id`**, not by region — matches the dominant "investigate this customer" query pattern; region kept as secondary index for compliance slicing.
4. **DB proxy (PgBouncer) mandatory** given horizontally-scaled stateless services — without it, connection counts blow out Postgres well before compute does.
5. **Event-sourced `case_events`** doubles as the audit trail — one mechanism satisfies both "how do I rebuild case state" and "prove to a regulator what happened and when."
6. **Cache invalidation is event-driven for anything analyst-facing-and-actionable** (queue view), TTL-based only for slowly-changing enrichment data — because staleness on the queue view has real fraud-operational cost, not just a UX cost.
7. **Zero-trust internal networking (mTLS/service mesh)** throughout, because this system is a high-value target (PII + PAN + fraud methodology) for both external attackers and insider threats.
