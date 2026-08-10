# Implementation Plan: Insurance Claim Photo + Statement Fraud Detector (Scale)

Companion to `systemdesign.md`. Phased so each stage ships something runnable and cost/quota risks surface early rather than at launch.

---

## Phase 0 — Foundations (Week 1–2)

**Goal:** GCP project, IAM, and data contracts in place before any pipeline code.

- Provision GCP project(s): consider separate quota pools for prod vs. a "surge" environment (see systemdesign §4.5).
- Enable APIs: Cloud Vision, Speech-to-Text V2, Natural Language, Vertex AI, Pub/Sub, Cloud Run, Cloud Tasks, Cloud Workflows, Firestore, Memorystore, BigQuery, Cloud KMS, Cloud Monitoring/Logging.
- Define GCS bucket layout and lifecycle policies (`raw-claims-bucket`, `processed-claims-bucket`) with Coldline transition rules.
- Define the claim data contract: `claim_id`, photo manifest (index, filename, hash fields to be filled), audio manifest, submission timestamp.
- Set up least-privilege service accounts per component (ingestion, vision-worker, speech-worker, aggregator).
- Request initial Vision/Speech quota increases sized to projected peak QPS (don't wait until load testing to discover the default quota is too low).

**Exit criteria:** a claim's raw files can be uploaded to GCS and trigger a Pub/Sub message end-to-end, with nothing downstream yet.

---

## Phase 1 — Ingestion & Async Backbone (Week 2–4)

**Goal:** synchronous upload path returns immediately; async fan-out works.

- Build ingestion endpoint (Cloud Run) that accepts claim submission, writes photos/audio to GCS, returns `202 Accepted` + `claim_id`.
- Configure Eventarc trigger on GCS finalize → Pub/Sub topic `claim-ingested`.
- Build Orchestrator (Cloud Workflows or a lightweight Cloud Run job) that:
  - Reads the claim manifest.
  - Computes SHA-256 + perceptual hash for each photo, audio hash, **before** any API call (pure local compute).
  - Writes hashes to Firestore.
  - Publishes to `vision-batch` and `speech-batch` topics with the claim_id and list of not-yet-cached items.
- Implement idempotency: Firestore doc per `claim_id/photo_index` with a processing-state field, checked before any downstream work.
- Set up dead-letter topics on every subscription with max-delivery-attempts.

**Exit criteria:** submitting a claim results in correctly fanned-out messages, with duplicate messages (simulate Pub/Sub redelivery) provably not double-processed.

---

## Phase 2 — Caching Layer (Week 3–5, parallel with Phase 1 tail)

**Goal:** avoid paying for Vision/Speech calls on content already seen.

- Stand up Memorystore (Redis) for hot hash lookups; Firestore as durable backing store.
- Implement near-duplicate matching (Hamming-distance threshold on pHash) in addition to exact-hash matching.
- Cache write-through: any Vision/Speech result gets written to Firestore keyed by hash, with 90-day TTL.
- Log a "cache_hit" fraud signal separately from the plain cost-optimization cache hit, so downstream scoring can use image/audio reuse as a feature.
- Build a small offline job against sample/historical data (if available) to estimate expected cache-hit rate — this number directly informs the cost model and should be validated, not assumed.

**Exit criteria:** re-submitting a known photo/audio hash skips the external API call and returns the cached annotation, measurably (via Cloud Monitoring counter: cache hits vs. misses).

---

## Phase 3 — Vision Worker with Tiered Feature Selection (Week 4–7)

**Goal:** cost-aware, batched image analysis.

- Implement Vision worker (Cloud Run, autoscaling, concurrency-limited to respect quota):
  - Pulls messages from `vision-batch`.
  - Groups photos into batches of up to 16 for `batchAnnotateImages`.
  - Tier 1 request: `LABEL_DETECTION` + `SAFE_SEARCH_DETECTION` on every non-cached photo.
  - Conditionally add `OBJECT_LOCALIZATION` when claim type suggests damage-area estimation is needed (e.g., vehicle/property damage claims vs. simple document claims).
- Implement escalation logic (Tier 2) as a separate consumer on a `vision-escalation` topic, triggered by rules evaluated on Tier 1 output (e.g., label suggests a document is present → trigger `DOCUMENT_TEXT_DETECTION`; suspicion score from aggregator crosses threshold → trigger custom tamper-detection model).
- Implement client-side token-bucket rate limiter tuned to negotiated quota; wire exponential backoff + jitter using the Vision client library's retry configuration (tune base delay, multiplier, cap, max attempts per systemdesign §4.5).
- Implement circuit breaker: track rolling error rate; pause message consumption on sustained 429/5xx.
- Write results to Firestore + `processed-claims-bucket`; publish to `features-ready`.

**Exit criteria:** load test at 2x expected peak QPS; verify quota headroom, backoff behavior under induced 429s (can be tested against a quota-limited test project), and that batch grouping is actually reducing call count as designed (verify via Cloud Monitoring API call counts vs. photo counts).

---

## Phase 4 — Speech Worker with Dynamic Batch (Week 5–7, parallel with Phase 3)

**Goal:** cost-optimized transcription pipeline.

- Implement Speech worker using Speech-to-Text V2 `BatchRecognize` with the Dynamic Batch tier as default.
- Route to standard (non-batch) tier only for a small, explicitly flagged subset (e.g., high-value or expedited claims) where the ~24-hour Dynamic Batch turnaround is unacceptable — implement this as a routing rule, not a global switch.
- Pipe transcripts through Natural Language API (entity extraction, sentiment) on 100% of transcripts.
- Implement the same backoff/circuit-breaker pattern as the Vision worker (shared library, don't reimplement per worker).
- Write transcript + NL features to Firestore; publish to `features-ready`.

**Exit criteria:** end-to-end transcript + entity/sentiment output available for a test claim within the expected Dynamic Batch SLA window; standard-tier fast-path verified separately.

---

## Phase 5 — Fraud Aggregator & Escalation to Vertex AI (Week 6–9)

**Goal:** combine photo + speech features into a fraud score; gate the expensive Gemini consistency check behind cheaper signals.

- Build aggregator (Cloud Run) subscribed to `features-ready`, waiting for both photo and speech features per claim (use Firestore state to know when a claim is "complete" — handle partial-failure cases, e.g., audio corrupted but photos fine).
- Implement the tiered custom fraud model (Vertex AI): combines label consistency (do photo labels match the claimed incident type), image-reuse signal, transcript sentiment/entity features, and (for escalated claims) tamper-detection output and Gemini consistency-check output.
- Implement escalation rule engine: define the concrete thresholds that route a claim from Tier 1 into the Gemini consistency check and/or custom tamper model — start with a conservative rule set and tune against labeled outcomes once available.
- Write final fraud score + supporting evidence to BigQuery (audit trail) and notify the claims management system via Pub/Sub.

**Exit criteria:** a claim flows end-to-end from ingestion to a fraud score with full evidence trail queryable in BigQuery by `claim_id`.

---

## Phase 6 — Observability, Cost Guardrails, Load Test (Week 8–10)

**Goal:** prove the system holds up at 50k claims/day and that cost tracks the model in `systemdesign.md`.

- Cloud Monitoring dashboards: ingestion rate, per-tier API call volume, cache-hit rate, quota utilization %, DLQ depth, cost-per-claim (computed from BigQuery cost-attribution table).
- Alerts: quota utilization > 70%, DLQ depth above threshold, cost-per-claim drift week-over-week beyond a defined tolerance.
- Load test: simulate 50k claims/day sustained plus a 5–10x burst window (catastrophe-event simulation) against a staging project with representative quota; confirm autoscaling, backoff, and circuit breakers behave as designed rather than cascading into quota exhaustion.
- Validate the cost model against actual billing export data after a burn-in period; recalibrate the Tier 1/Tier 2 escalation thresholds if actual escalation rate diverges materially from the assumed ~10–15%.

**Exit criteria:** sustained load test passes without quota breaches or unbounded DLQ growth; measured cost-per-claim is within an agreed tolerance of the modeled estimate.

---

## Phase 7 — Security, Compliance, Rollout (Week 9–11)

- CMEK encryption review for PII-bearing buckets/collections (photos, audio, transcripts).
- IAM audit: confirm least-privilege service accounts, no worker has broader access than its function requires.
- Data retention/deletion policy implemented via GCS lifecycle rules aligned to applicable regulatory retention windows.
- Staged rollout: shadow mode (score claims but don't act on the score) → limited region/claim-type rollout → full rollout, with a rollback path (feature flag to fall back to fully manual review) at each stage.

---

## Cross-Cutting Engineering Practices (apply throughout, not a separate phase)

- **Idempotency everywhere**: every consumer must tolerate Pub/Sub at-least-once redelivery without duplicate billing or duplicate fraud scores.
- **Backoff/retry as a shared library**, not reimplemented per worker — one place to tune base delay, multiplier, jitter, max attempts, and circuit-breaker thresholds.
- **Quota headroom as a release gate**: no new feature that adds Vision/Speech/Vertex AI call volume ships without first checking current quota utilization and, if needed, requesting an increase.
- **Cost-per-claim as a tracked metric from day one** in BigQuery, not retrofitted after a billing surprise.

---

## Rough Timeline Summary

| Phase | Weeks | Key deliverable |
|---|---|---|
| 0 — Foundations | 1–2 | Project/IAM/quota setup |
| 1 — Ingestion & async backbone | 2–4 | Claim upload → fan-out working |
| 2 — Caching layer | 3–5 | Dedupe cache live, hit-rate measured |
| 3 — Vision worker | 4–7 | Tiered, batched, quota-safe Vision pipeline |
| 4 — Speech worker | 5–7 | Dynamic Batch transcription pipeline |
| 5 — Fraud aggregator | 6–9 | End-to-end fraud score with evidence trail |
| 6 — Observability & load test | 8–10 | Verified at 50k claims/day + burst |
| 7 — Security & rollout | 9–11 | Shadow → staged → full production |

(Phases overlap deliberately — Vision and Speech workers, and caching, can be built in parallel once Phase 1's backbone exists.)
