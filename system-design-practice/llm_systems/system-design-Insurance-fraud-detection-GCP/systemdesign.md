# System Design: Insurance Claim Photo + Statement Fraud Detector (Scale)

## 1. Problem Restatement & Scale Envelope

- **Volume:** 50,000 claims/day
- **Photos:** 3–10 per claim → ~325,000 photos/day (avg 6.5), bursty around business hours
- **Audio:** 1 recorded phone statement per claim → 50,000 audio files/day (assume 2–5 min avg → ~150,000–250,000 minutes/day)
- **Design axis:** cost + throughput, not just correctness. Every design decision below is justified against a $ and req/sec tradeoff, not just accuracy.

Back-of-envelope average load: 325,000 images / 86,400 sec ≈ **3.8 images/sec average**, 50,000 audio files/day ≈ **0.6 files/sec average**. Peak-to-average ratio for claims intake is typically 4–6x (business hours, post-storm/catastrophe spikes), so we design for ~20–25 images/sec sustained peak, with the ability to absorb 10x catastrophe-event bursts via queuing rather than compute scaling alone.

This is fundamentally a **data pipeline / batch-and-stream hybrid problem**, not a request/response API problem. Treating it as synchronous request/response is the single biggest anti-pattern to avoid.

---

## 2. Non-Goals / Explicit Assumptions

- Real-time (sub-second) fraud verdicts are **not** required. Claims adjusters work on an hours-to-next-day SLA, not milliseconds. This assumption unlocks batching, async queues, and Dynamic Batch pricing tiers everywhere.
- We assume claim photos/audio arrive via a mobile app or agent portal upload, landing first in object storage — not streamed frame-by-frame.
- We assume a downstream fraud-scoring model (custom, on Vertex AI) consumes structured features (labels, damage estimates, transcript, tamper signals) rather than raw bytes — Vision/Speech APIs are a **feature-extraction layer**, not the fraud model itself.

---

## 3. High-Level Architecture (ASCII)

```
                                   ┌────────────────────────────┐
                                   │   Claims Intake (mobile /   │
                                   │   agent portal / web)       │
                                   └──────────────┬──────────────┘
                                                  │ upload photos+audio
                                                  ▼
                                   ┌────────────────────────────┐
                                   │   Cloud Storage (GCS)       │
                                   │   raw-claims-bucket/        │
                                   │     {claim_id}/photo_*.jpg  │
                                   │     {claim_id}/stmt.wav     │
                                   └──────────────┬──────────────┘
                                                  │ GCS finalize event
                                                  ▼
                                   ┌────────────────────────────┐
                                   │   Eventarc → Pub/Sub topic  │
                                   │   "claim-ingested"          │
                                   └──────────────┬──────────────┘
                                                  ▼
                                   ┌────────────────────────────┐
                                   │  Orchestrator (Cloud        │
                                   │  Workflows / Cloud Run job) │
                                   │  - dedupe check (perceptual │
                                   │    hash against Firestore)  │
                                   │  - fan-out sub-tasks        │
                                   └───────┬───────────┬────────┘
                                           │           │
                     ┌─────────────────────┘           └────────────────────┐
                     ▼                                                     ▼
      ┌───────────────────────────┐                        ┌───────────────────────────┐
      │ Pub/Sub: "vision-batch"   │                        │ Pub/Sub: "speech-batch"   │
      └──────────────┬────────────┘                        └──────────────┬────────────┘
                     ▼                                                     ▼
      ┌───────────────────────────┐                        ┌───────────────────────────┐
      │ Vision Worker (Cloud Run, │                        │ Speech Worker (Cloud Run, │
      │ autoscaling, batched      │                        │ Speech-to-Text V2 async / │
      │ batchAnnotateImages,      │                        │ Dynamic Batch Recognize)  │
      │ tiered feature selection) │                        │                           │
      └──────────────┬────────────┘                        └──────────────┬────────────┘
                     │  cache check/write                                  │
                     ▼                                                     ▼
      ┌───────────────────────────┐                        ┌───────────────────────────┐
      │ Memorystore (Redis) /     │                        │ Firestore: transcript +   │
      │ Firestore: image-hash →   │                        │ NL API entities/sentiment │
      │ cached Vision result      │                        └──────────────┬────────────┘
      └──────────────┬────────────┘                                       │
                     │                                                    │
                     └──────────────┬─────────────────────────────────────┘
                                    ▼
                     ┌────────────────────────────────┐
                     │ Pub/Sub: "features-ready"       │
                     └───────────────┬────────────────┘
                                     ▼
                     ┌────────────────────────────────┐
                     │ Fraud Aggregator / Scoring      │
                     │ (Cloud Run + Vertex AI custom   │
                     │ model: photo-statement          │
                     │ consistency, tamper score,      │
                     │ damage-estimate cross-check)    │
                     └───────────────┬────────────────┘
                                     ▼
                     ┌────────────────────────────────┐
                     │ BigQuery (audit/analytics) +    │
                     │ Claims System notification      │
                     │ (Pub/Sub → claims-mgmt API)     │
                     └────────────────────────────────┘

     Cross-cutting:
     - Cloud Tasks: scheduled retries w/ exponential backoff for any failed API call
     - Dead-letter Pub/Sub topics on every subscription (max delivery attempts)
     - Cloud Monitoring: quota-usage dashboards, alert on 429 rate > threshold
     - Cloud KMS: encryption of PII (audio contains voice/PII, photos may contain faces/plates)
```

---

## 4. Key Design Decisions (the parts an interviewer is probing for)

### 4.1 Async job queue, not synchronous request/response

Claim submission returns immediately (HTTP 202 + claim_id) after the raw files land in GCS. All Vision/Speech/scoring work happens via **Pub/Sub-triggered workers**, decoupled from the upload path. This is essential because:

- Vision/Speech calls have variable latency (hundreds of ms to seconds) and 50k claims/day × up to 10 photos means a synchronous path would need to hold open ~3.8 concurrent long-lived connections *on average* but 10-20x that at peak — wasteful and fragile.
- Pub/Sub + Cloud Run (autoscaling on queue depth) lets us decouple ingestion rate from processing rate, absorbing catastrophe-event bursts (e.g., hailstorm → 5x claims in an hour) without over-provisioning for the average case.
- Failures are retried at the message level (Pub/Sub redelivery + Cloud Tasks backoff) instead of failing an entire user-facing request.

### 4.2 Batching Vision and Speech calls

- **Vision API `batchAnnotateImages`**: bundles up to 16 image requests into a single API call. Instead of issuing one HTTP call per photo (325k calls/day), the Vision worker batches all photos for a claim (3–10) into 1 request, reducing call count by up to 10x and cutting per-request overhead (connection setup, auth, quota-unit consumption is still per-image, but *call count* against QPS quotas drops sharply).
- **Speech-to-Text V2 `BatchRecognizeRequest`**: submit multiple audio files in one batch job that writes transcripts back to GCS asynchronously, rather than one long-running-operation per file issued synchronously. This is the correct API shape for "not time-sensitive" workloads.
- Net effect: fewer HTTP round trips, fewer TCP/TLS handshakes, and — critically — far less exposure to per-minute QPS quota limits (see §4.4), since quota is generally enforced on request count/QPS as well as unit volume.

### 4.3 Caching repeated API calls

Fraud rings frequently **reuse the same photos across multiple claims** (stock damage photos, recycled staged-accident images) and sometimes the same claimant submits duplicate images across a claim. This is both a cost optimization and a fraud signal.

- Compute a **perceptual hash (pHash)** + exact **SHA-256** for every incoming photo at ingestion time (cheap, local compute, no API call).
- Look up the hash in **Memorystore (Redis)** with a fallback to **Firestore** for cold entries. If a hash (or near-duplicate pHash within a Hamming-distance threshold) has been seen before:
  - Reuse the cached Vision annotation result — **skip the Vision API call entirely** (this directly reduces billable units).
  - Flag the claim with an "image reuse" fraud signal, since an image reused across unrelated claims is itself suspicious. This is a case where the cache lookup is *both* a cost saver and a fraud feature — worth calling out explicitly.
- Same pattern for audio: hash the audio file; exact duplicate phone statements across claims (a "recorded script" fraud pattern) are both cached and flagged.
- Cache TTL: 90 days (matches typical claim review window), with LRU eviction in Redis and durable copy in Firestore for audit/compliance retention.

### 4.4 Cheaper feature sets / tiered analysis (accuracy-vs-cost tradeoff)

Not every photo needs every Vision feature. Vision API bills **per feature per image** (e.g., Label Detection + Object Localization on the same image = 2 billable units), so feature selection directly multiplies cost.

**Tier 1 — cheap pass, applied to 100% of photos:**
- `LABEL_DETECTION` (~$1.50/1000 units in the standard tier, dropping to $1.00/1000 above 5M/month) — broad scene/object categorization (car, dent, fire, water damage, etc.)
- `SAFE_SEARCH_DETECTION` — **free when bundled with Label Detection** — screens for policy-violating content.
- `OBJECT_LOCALIZATION` only where bounding boxes matter (e.g., estimating damage area) — priced higher (~$2.25/1000) so applied selectively, not universally.

**Tier 2 — escalate only flagged claims (~10–15% expected, tunable):**
- `DOCUMENT_TEXT_DETECTION` if a photo contains a receipt/invoice/repair estimate — more expensive OCR, only run when Tier 1 labels suggest a document is present.
- Custom Vertex AI vision model for **tamper/forgery detection** (error-level analysis, metadata/EXIF consistency, splice detection) — Vision API has no native forgery-detection feature, so this is a purpose-built model invoked only for claims where Tier 1 + statement analysis already raised suspicion. This avoids running an expensive custom-model inference on every one of 325k photos/day.
- Reverse-image / `WEB_DETECTION` ($3.50/1000, the most expensive Vision feature) only for claims flagged as high-value or where duplicate-hash detection didn't catch a near-match — used sparingly given cost.

**Speech side, same tiering:**
- Tier 1: `Speech-to-Text V2` transcription using the **Dynamic Batch** tier (~$0.003–0.004/min vs. ~$0.016/min for standard — roughly 4–5x cheaper) for all statements, since transcription is never time-critical here. Follow with **Natural Language API** entity extraction and sentiment (cheap, per-doc pricing) on 100% of transcripts.
- Tier 2: only for claims flagged by Tier 1 (label mismatch, sentiment/consistency anomaly), send transcript + claim narrative to a Vertex AI Gemini call for deeper cross-statement consistency checking (does the verbal account match the photo evidence timeline?). This is the most expensive step per-unit, so it's gated behind cheaper signals, not run universally.

This tiering is the main lever for controlling unit economics at 50k claims/day — it turns an "always run everything" bill into "run cheap universal screens, escalate the minority."

### 4.5 API quota limits and backoff/retry (first-class, not an afterthought)

At this scale, quota exhaustion is a near-certainty without explicit design:

- **Request GCP quota increases proactively** for Vision API (default is per-minute request quotas, project-level) and Speech-to-Text concurrent recognition quota, sized to peak QPS (not average), with headroom for catastrophe bursts.
- **Client-side rate limiting / token bucket** in the Vision/Speech workers to stay under negotiated quota even if Cloud Run autoscaling tries to scale out further than the quota allows — otherwise autoscaling just produces a wall of 429s.
- **Exponential backoff with jitter** on every Vision/Speech call: e.g., base 1s, 2x multiplier, capped at 60s, ±20% jitter, max 5–7 attempts, using the client libraries' built-in retry configuration where possible.
- **Circuit breaker** at the worker level: if error rate from Vision/Speech exceeds a threshold (e.g., sustained 429/5xx over a rolling window), stop pulling new Pub/Sub messages for a cool-down period rather than hammering the API and burning through retry budget — messages stay safely queued (Pub/Sub retains unacked messages) rather than being dropped.
- **Dead-letter topics** on every Pub/Sub subscription with a max-delivery-attempts setting, so permanently-failing messages (malformed image, corrupt audio) land in a DLQ for manual triage instead of retrying forever or silently vanishing.
- **Separate quota pools / GCP projects per environment or by traffic class** (e.g., isolate a "catastrophe surge" project or use quota reservations) so a spike doesn't starve routine daily processing.
- Monitor **quota utilization %** as a first-class Cloud Monitoring metric with alerting well before hitting 100% (e.g., alert at 70%), not just alerting on outright failures.

### 4.6 Cost model (illustrative, current published GCP rates)

| Component | Assumption | Rate | Daily cost (approx) |
|---|---|---|---|
| Vision Label Detection (Tier 1, all photos) | 325,000 units/day | $1.50/1000 (mid-tier) | ~$488 |
| Vision Safe Search | bundled free with Label Detection | $0 | $0 |
| Vision Object Localization (selective, ~30% of photos) | ~97,500 units/day | $2.25/1000 | ~$219 |
| Vision escalation tier (Doc Text / Web Detection, ~10% of photos) | ~32,500 units/day | ~$2–3.50/1000 blended | ~$90 |
| Speech-to-Text Dynamic Batch (all statements) | ~200,000 min/day | $0.003–0.004/min | ~$600–800 |
| Natural Language API (entities/sentiment, all transcripts) | 50,000 docs/day | low per-doc rate | ~$50–100 |
| Vertex AI escalation (Gemini consistency check, ~12% of claims) | 6,000 calls/day | model-dependent | variable, budgeted separately |
| **Vision + Speech subtotal (excl. Vertex AI escalation)** | | | **~$1,450–1,700/day** |

Cache hits (duplicate/reused images and statements) reduce the effective billable volume below these figures — the exact discount depends on measured duplicate rates in production, but even a 10–15% cache-hit rate is a meaningful, ongoing cost reduction at this volume, which is why caching is worth building on day one rather than as a later optimization.

Note: pricing figures reflect currently published GCP rates and should be re-verified against `https://cloud.google.com/vision/pricing` and `https://cloud.google.com/speech-to-text/pricing` before finalizing a budget, since cloud pricing changes over time.

---

## 5. Data & Storage Layer

- **Cloud Storage**: raw photos/audio (`raw-claims-bucket`), processed/annotated outputs (`processed-claims-bucket`), lifecycle policy to move to Coldline/Archive after the active-claim window (e.g., 90 days) for cost control, since regulatory retention windows for insurance claims are typically multi-year.
- **Firestore**: claim metadata, dedupe hash index, cached Vision/Speech results, workflow state (idempotency keys per claim_id + photo index to avoid double-processing on Pub/Sub redelivery).
- **Memorystore (Redis)**: hot-path cache for hash lookups (low-latency dedupe check before deciding whether to call Vision at all).
- **BigQuery**: append-only audit log of every API call, cost, quota event, and fraud score, partitioned by day — feeds both cost dashboards and model retraining/evaluation.

## 6. Idempotency & Exactly-Once-Enough Processing

Pub/Sub is at-least-once delivery. Every worker must be idempotent:
- Use `claim_id + photo_index` (or content hash) as an idempotency key written to Firestore before processing; workers check-and-skip if already processed.
- Vision/Speech batch job outputs written to GCS with deterministic paths (`{claim_id}/vision-result.json`) so re-running a batch job overwrites rather than duplicates.

## 7. Observability

- Cloud Monitoring dashboards: claims ingested/hour, Vision/Speech QPS vs. quota, cache hit rate, cost-per-claim (rolling), DLQ depth, escalation-tier trigger rate.
- Structured logging (Cloud Logging) with `claim_id` as a correlation ID across every service hop, so a single claim's full processing trace (ingestion → vision → speech → fraud score) is queryable.
- Alerting: quota utilization > 70%, DLQ depth > threshold, cost-per-claim drift > X% week-over-week (catches a tiering misconfiguration or feature-selection bug before it becomes a budget surprise).

## 8. Security & Compliance Notes

- Photos may contain faces, license plates, addresses; audio contains voice biometric + PII narrative. Encrypt at rest (default GCS/Firestore encryption, optionally CMEK via Cloud KMS) and in transit.
- Access control via IAM least-privilege service accounts per worker (Vision worker cannot read Speech results bucket, etc.).
- Retention/deletion policy aligned to insurance regulatory requirements (varies by jurisdiction) — implemented via GCS Object Lifecycle Management, not manual cleanup.

## 9. What I'd explicitly flag as open questions in an interview

- Exact duplicate/cache-hit rate is an empirical question — needs a pilot to size the real cost savings.
- Where the Tier 1 → Tier 2 escalation threshold should sit is a precision/recall/cost tradeoff that needs labeled fraud data to tune, not a guess.
- Whether Dynamic Batch's 24-hour turnaround SLA is acceptable end-to-end, or whether some claim types (e.g., high-value/total-loss) need the faster standard Speech tier despite the higher cost — likely a per-claim-value routing decision, not a global setting.
