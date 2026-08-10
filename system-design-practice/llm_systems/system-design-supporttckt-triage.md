# System Design: Multi-Channel Support Ticket Normalization & Triage Pipeline

## 1. Problem Statement

Support tickets arrive in three raw forms:

- **Text** (email, chat, web form)
- **Voicemail** (audio file, any language)
- **Screenshot** (image containing text — error dialogs, chat screenshots, etc.)

Goal: normalize all three into a single canonical ticket object that is:

1. English-language text
2. Scored for sentiment
3. Tagged with extracted entities
4. Flagged for urgency
5. Ready to route into the triage queue

## 2. Core Design Decision: API Chaining Order

This is the crux of the design, and it's easy to get wrong.

```
Voicemail (.wav/.mp3)   Screenshot (.png/.jpg)   Text (any language)
        |                        |                        |
        v                        v                        |
  Speech-to-Text             Vision API                    |
  (STT)                      (OCR — TEXT_DETECTION,        |
        |                    NOT label/object detection)   |
        |                        |                         |
        +------------------------+-------------------------+
                                  |
                                  v
                       Raw text, source language unknown
                                  |
                                  v
                        Translation API
                (detect language -> translate to English)
                        MUST run before NL API
                                  |
                                  v
                       English-normalized text
                                  |
                                  v
                        Natural Language API
              (sentiment analysis + entity extraction
               + optionally entity sentiment)
                                  |
                                  v
                  Urgency scoring (rules + NL signals)
                                  |
                                  v
                      Canonical Triaged Ticket (JSON)
                                  |
                                  v
                        Triage queue / ticketing system
```

### Why Translation must run *before* Natural Language

Google's Natural Language API's pre-trained sentiment and entity models are tuned predominantly on English text. Running sentiment analysis on non-English text either:

- silently degrades accuracy (model does its best-effort but confidence is unreliable), or
- is simply unsupported for the specific feature requested for that language.

Translating to English first gives one consistent language for the sentiment/entity layer downstream, and lets every ticket — regardless of source language — be triaged against the same rules and the same historical baseline (e.g., "average sentiment score for a P1 ticket").

**Common mistake to watch for:** running Natural Language directly on OCR'd or transcribed text without a language-detection + translation step, then wondering why sentiment scores for non-English tickets are noisy or flat.

### Why Vision is OCR here, not object/label detection

The Vision API is a general vision API with many feature types: `LABEL_DETECTION`, `OBJECT_LOCALIZATION`, `FACE_DETECTION`, `SAFE_SEARCH_DETECTION`, `TEXT_DETECTION`, `DOCUMENT_TEXT_DETECTION`, etc. For this pipeline we only ever need:

- `TEXT_DETECTION` — good for short strings, UI elements, sparse text (typical for a screenshot of an error toast or a chat bubble)
- `DOCUMENT_TEXT_DETECTION` — better for dense text, paragraphs, or screenshots of emails/documents; uses different internal OCR tuned for dense layouts and returns structured `page/block/paragraph/word` hierarchy plus better handling of line breaks

**Design choice:** call `DOCUMENT_TEXT_DETECTION` by default (it is a superset — works well on both dense and sparse text) rather than plain `TEXT_DETECTION`, unless we want the raw per-word bounding-box array that plain `TEXT_DETECTION` returns first.

We explicitly do **not** use label/object detection — a screenshot ticket isn't "what objects appear in this image," it's "what does the text say."

## 3. Component Breakdown

### 3.1 Ingestion Layer
- Cloud Storage bucket(s): `tickets-raw-audio`, `tickets-raw-images`, `tickets-raw-text`
- Pub/Sub topic `ticket-ingested` fires on object finalize (via Cloud Storage notifications) or on direct text-channel submission (API Gateway / Cloud Run endpoint)
- Each message carries: `ticket_id`, `channel` (`text|voice|image`), `gcs_uri` (if applicable), `received_at`

### 3.2 Channel Normalizers (Cloud Run services / Cloud Functions, one per channel, triggered by Pub/Sub)

| Channel | Service | Primary GCP API | Notes |
|---|---|---|---|
| Voicemail | `stt-normalizer` | Speech-to-Text v2 | Use async `LongRunningRecognize` for anything >1 min; enable `automatic_language_detection` or `language_code` alternatives list; enable punctuation and speaker diarization if multi-speaker |
| Screenshot | `ocr-normalizer` | Vision API `DOCUMENT_TEXT_DETECTION` | Handle multi-image tickets (screenshot bursts) by concatenating in upload order |
| Text | `text-passthrough` | — | Ticket text passed straight through; language not assumed |

All three normalizers write to a common intermediate schema and publish to `ticket-normalized` topic:

```json
{
  "ticket_id": "T-10293",
  "channel": "voice",
  "raw_extracted_text": "...",
  "source_confidence": 0.94,
  "detected_language_hint": null
}
```

### 3.3 Language Normalizer (`translate-normalizer`, Cloud Run)
- Input: `raw_extracted_text`
- Step 1: Cloud Translation API `detectLanguage` (skip if already tagged `en` with high confidence from STT)
- Step 2: If not English, `translateText` -> English
- Output: `normalized_text_en`, `source_language`, retains `raw_extracted_text` for audit/original-language display in the UI

### 3.4 NLU Layer (`nlu-processor`, Cloud Run)
- Input: `normalized_text_en`
- Calls Cloud Natural Language API:
  - `analyzeSentiment` -> document-level sentiment `score` (-1.0 to 1.0) and `magnitude`
  - `analyzeEntities` (or `analyzeEntitySentiment` for per-entity sentiment, e.g., isolating anger toward a specific product/feature vs. the company generally)
- Output appended to ticket record.

### 3.5 Urgency Scoring (`urgency-scorer`, Cloud Function, rules engine)

Urgency is **not** purely an NL API output — it's a composite score combining:

1. **Sentiment score** from NL API (strong negative sentiment raises urgency)
2. **Magnitude** from NL API (high magnitude = strong emotion regardless of polarity, could indicate distress)
3. **Keyword/entity rules**: presence of entities or terms like "outage," "data loss," "cannot log in," "payment failed," "legal," "security," "breach" (a small curated keyword list or a lightweight classifier)
4. **Channel signal**: voicemail tickets get a slight urgency bump by default (a customer who calls rather than emails is statistically more likely to be blocked/urgent)
5. **SLA/account metadata** (if available from CRM): enterprise tier accounts get priority weighting

Output: `urgency: {LOW|MEDIUM|HIGH|CRITICAL}` plus the underlying `urgency_score` (0-100) for tunability, and `urgency_reasons: []` for explainability (important for support agent trust in the auto-triage).

### 3.6 Canonical Ticket Store & Routing
- Final merged record written to Firestore (or BigQuery for analytics + Firestore for the live queue)
- Pub/Sub `ticket-triaged` topic notifies the ticketing system (Zendesk/Jira Service Desk/custom) via a webhook consumer
- BigQuery sink (via Pub/Sub -> BigQuery subscription) for reporting/dashboards (sentiment trends, urgency distribution over time)

## 4. Canonical Ticket Schema

```json
{
  "ticket_id": "T-10293",
  "channel": "voice",
  "received_at": "2026-07-22T09:14:00Z",
  "source_language": "es",
  "raw_extracted_text": "No puedo acceder a mi cuenta y perdi todos mis datos...",
  "normalized_text_en": "I can't access my account and I lost all my data...",
  "sentiment": {
    "score": -0.8,
    "magnitude": 4.2
  },
  "entities": [
    {"name": "account", "type": "OTHER", "salience": 0.31},
    {"name": "data", "type": "OTHER", "salience": 0.22}
  ],
  "urgency": "CRITICAL",
  "urgency_score": 91,
  "urgency_reasons": ["strong_negative_sentiment", "high_magnitude", "keyword:data loss", "channel:voice"],
  "processing_trace": {
    "stt_confidence": 0.94,
    "ocr_confidence": null,
    "translation_confidence": 0.97
  }
}
```

## 5. GCP Services Used

| Concern | Service |
|---|---|
| Voicemail transcription | Speech-to-Text v2 (async, LongRunningRecognize) |
| Screenshot OCR | Vision API — `DOCUMENT_TEXT_DETECTION` |
| Non-English -> English | Cloud Translation API (Advanced, v3, for glossary/custom-model support if needed later) |
| Sentiment + entities | Cloud Natural Language API |
| Compute / orchestration | Cloud Run (stateless normalizer services) + Cloud Functions (lightweight rules step) |
| Messaging/event backbone | Pub/Sub |
| Storage (raw media) | Cloud Storage |
| Storage (canonical tickets, live queue) | Firestore |
| Analytics | BigQuery (via Pub/Sub subscription sink) |
| Orchestration alternative | Workflows or Cloud Composer if the pipeline needs branching/retries beyond what Pub/Sub chaining gives cleanly |
| Observability | Cloud Logging + Cloud Monitoring + Error Reporting |
| Secrets/config | Secret Manager (API keys not needed since these are IAM-authenticated GCP APIs, but any 3rd-party ticketing webhook secrets go here) |

## 6. Orchestration Pattern: Pub/Sub Chaining vs. Workflows

Two viable patterns:

**A. Choreography (Pub/Sub chaining)** — each stage publishes to the next topic; simplest to build, scales independently, but harder to see the full trace of one ticket without stitching logs together.

**B. Orchestration (Cloud Workflows or a single Cloud Run service with sequential calls)** — a single Workflow definition explicitly calls STT/Vision -> Translation -> NL -> Urgency step by step, with native retry/error-handling per step and a visual execution history per ticket.

**Recommendation:** Use **Workflows** for this pipeline. Ticket volume is unlikely to be so extreme that per-stage independent scaling matters more than debuggability, and support/triage systems benefit heavily from being able to inspect "exactly what happened to ticket T-10293" as a single execution trace — which choreography makes annoying. Pub/Sub is still used at the ingestion boundary (fan-in from three channels) and at the egress boundary (fan-out to ticketing system + BigQuery).

## 7. Error Handling & Edge Cases

- **Low-confidence OCR/STT**: if `source_confidence` < threshold (e.g., 0.6), flag ticket as `needs_human_review` rather than silently triaging on garbage text.
- **Multi-language / code-switched text**: Translation API detects the dominant language; mixed-language tickets are translated as a whole rather than segmented (acceptable tradeoff for v1).
- **Empty/garbled OCR result** (e.g., screenshot of a photo with no text): fall back to routing ticket as `UNCLASSIFIED — image attachment, manual review`.
- **Long voicemails**: use async recognition; set a max duration (e.g., 5 min) and truncate/flag beyond that.
- **API quota/rate limits**: Workflows retry policy with exponential backoff; dead-letter topic for tickets that fail after N retries, surfaced to an ops dashboard.
- **PII in voicemail/screenshots**: consider Cloud DLP API as an optional stage before storage/logging, especially since screenshots often contain account numbers, emails, etc.

## 8. Cost & Scaling Notes

- Speech-to-Text and Vision are priced per unit of audio/image processed — batch low-priority tickets where possible.
- Natural Language and Translation are priced per character/unit — since text is normalized once and reused, avoid re-calling these APIs on retries unless the upstream text actually changed.
- Cloud Run scales to zero, appropriate for uneven support ticket volume (e.g., spikes during incidents).
- BigQuery costs driven by streaming inserts — consider batching every N seconds instead of one row per ticket if volume is very high.

## 9. What Good Looks Like (Validation Checklist)

- [ ] A Spanish voicemail complaining about data loss ends up as an English ticket with negative sentiment and CRITICAL urgency
- [ ] A screenshot of a Japanese error dialog is OCR'd, translated, and triaged — not misrouted through label/object detection
- [ ] Sentiment scores are only ever computed on the English-normalized text, never on raw multilingual text
- [ ] Low-confidence transcriptions/OCR are flagged for human review instead of confidently mis-triaged
- [ ] Every ticket, regardless of channel, lands in the same canonical schema in Firestore/BigQuery

--
# Implementation Plan: Multi-Channel Support Ticket Normalization & Triage Pipeline

Companion to `systemdesign.md`. This plan sequences the build into phases, each independently demoable, so the OCR-vs-object-detection and translation-before-NL decisions get validated early rather than discovered late.

## Phase 0 — Project Setup (0.5 day)

- [ ] Create/confirm GCP project, billing enabled
- [ ] Enable APIs:
  ```
  gcloud services enable \
    speech.googleapis.com \
    vision.googleapis.com \
    translate.googleapis.com \
    language.googleapis.com \
    run.googleapis.com \
    workflows.googleapis.com \
    pubsub.googleapis.com \
    firestore.googleapis.com \
    bigquery.googleapis.com \
    storage.googleapis.com
  ```
- [ ] Create Cloud Storage buckets: `tickets-raw-audio`, `tickets-raw-images`
- [ ] Create Firestore database (native mode) for canonical ticket store
- [ ] Create BigQuery dataset `ticket_analytics` with table `triaged_tickets`
- [ ] Create service account `ticket-pipeline-sa` with roles: `roles/speech.editor`, `roles/vision.editor` (or `apiUser`), `roles/translate.user`... actually use `roles/cloudtranslate.user`, `roles/language.editor`... use `roles/language.aiPlatformUser` — verify exact role names at deploy time — plus `roles/pubsub.editor`, `roles/datastore.user`, `roles/bigquery.dataEditor`, `roles/workflows.invoker`

## Phase 1 — Prove the Two Trick Points in Isolation (1 day)

Before building the full pipeline, validate the two things most likely to be misunderstood:

### 1a. Vision OCR, not object detection
- [ ] Write a standalone script that calls `DOCUMENT_TEXT_DETECTION` (not `LABEL_DETECTION`) against 3-5 sample support screenshots (error dialogs, chat screenshots, a dense email screenshot)
- [ ] Confirm output is the actual text string, not object/label tags
- [ ] Compare `TEXT_DETECTION` vs `DOCUMENT_TEXT_DETECTION` on a dense-text sample to confirm the latter handles paragraph structure better

### 1b. Translation before Natural Language
- [ ] Take a non-English sample ticket (e.g., Spanish complaint text)
- [ ] Run Natural Language `analyzeSentiment` on it directly (untranslated) and record the result
- [ ] Translate to English via Translation API, then run `analyzeSentiment` again
- [ ] Compare — this becomes the justification artifact for why translation is a mandatory upstream stage, not optional

**Exit criterion:** a short internal note/demo showing both comparisons, confirming the pipeline order in `systemdesign.md`.

## Phase 2 — Channel Normalizers (2-3 days)

### 2a. Text channel (trivial passthrough)
- [ ] Cloud Run service `text-passthrough`: accepts ticket text via HTTP, writes to `ticket-normalized` schema, publishes to Pub/Sub topic `ticket-normalized`

### 2b. Voicemail (Speech-to-Text)
- [ ] Cloud Run service `stt-normalizer`, triggered by Cloud Storage finalize event on `tickets-raw-audio`
- [ ] Use `LongRunningRecognize` (v2 API) with `language_code: "auto"` or an explicit alternative-language list based on expected customer base
- [ ] Enable automatic punctuation
- [ ] Store `source_confidence` from the STT response
- [ ] Publish normalized record to `ticket-normalized`

### 2c. Screenshot (Vision OCR)
- [ ] Cloud Run service `ocr-normalizer`, triggered by Cloud Storage finalize event on `tickets-raw-images`
- [ ] Call `DOCUMENT_TEXT_DETECTION`
- [ ] Handle multi-image tickets: if a ticket has multiple screenshots, concatenate extracted text in upload order with a separator
- [ ] Publish normalized record to `ticket-normalized`

**Exit criterion:** submitting a sample file/text of each channel type produces a row in an intermediate `ticket-normalized` log/table with correctly extracted raw text.

## Phase 3 — Language Normalization (1 day)

- [ ] Cloud Run service `translate-normalizer`, subscribed to `ticket-normalized`
- [ ] Call `detectLanguage`; skip translation call if already English with high confidence
- [ ] Call `translateText` -> English otherwise
- [ ] Preserve both `raw_extracted_text` (original language) and `normalized_text_en`
- [ ] Publish to `ticket-translated`

**Exit criterion:** a Spanish, Japanese, and English sample ticket all emerge with `normalized_text_en` populated correctly.

## Phase 4 — NLU Layer (1 day)

- [ ] Cloud Run service `nlu-processor`, subscribed to `ticket-translated`
- [ ] Call `analyzeSentiment` on `normalized_text_en` — document-level score + magnitude
- [ ] Call `analyzeEntitySentiment` for per-entity sentiment (covers both entity extraction and per-entity polarity in one call)
- [ ] Publish to `ticket-analyzed`

**Exit criterion:** the same 3 sample tickets from Phase 3 now carry sentiment scores and entity lists.

## Phase 5 — Urgency Scoring & Canonical Store (1-2 days)

- [ ] Cloud Function `urgency-scorer`, subscribed to `ticket-analyzed`
- [ ] Implement scoring rules (start simple, tune later):
  - Base score from `abs(sentiment.score) * magnitude` scaled to 0-100
  - Keyword rule bonus (curated list: outage, breach, data loss, cannot login, payment failed, security, legal)
  - Channel bonus for voice
  - Confidence penalty: if `source_confidence` < 0.6, force `needs_human_review = true` regardless of computed urgency
- [ ] Map numeric score to bucket: `LOW < 30`, `MEDIUM < 60`, `HIGH < 85`, `CRITICAL >= 85` (tune thresholds against real data in Phase 7)
- [ ] Write final canonical record to Firestore collection `triaged_tickets`
- [ ] Stream same record to BigQuery via Pub/Sub -> BigQuery subscription

**Exit criterion:** a fully triaged ticket document exists in Firestore matching the canonical schema in `systemdesign.md`, and the same row appears in BigQuery.

## Phase 6 — Orchestration Consolidation (1-2 days)

- [ ] Decide: keep Pub/Sub choreography (Phases 2-5 as-is) or migrate to a single Cloud Workflows definition calling each step sequentially with built-in retry/error branches (recommended in systemdesign.md for debuggability)
- [ ] If migrating to Workflows: define YAML/JSON workflow with steps `stt_or_ocr -> translate -> nlu -> urgency -> persist`, conditional branch on `channel`
- [ ] Add dead-letter topic + alerting for tickets that fail after N retries at any stage

**Exit criterion:** one execution trace per ticket, viewable end-to-end in Cloud Console (Workflows execution history), covering happy path and at least one forced-failure path.

## Phase 7 — Routing, Dashboards, Tuning (2-3 days)

- [ ] Webhook consumer to push triaged tickets into the actual ticketing system (Zendesk/Jira/custom queue) based on `urgency` bucket
- [ ] Looker Studio or simple BigQuery-backed dashboard: sentiment distribution, urgency distribution, volume by channel, average time-to-triage
- [ ] Tune urgency thresholds and keyword list against a batch of historical tickets (if available) or a manually-labeled validation set of ~50-100 tickets
- [ ] Add Cloud DLP scan as optional pre-storage step if PII exposure in raw voicemail/screenshot text is a concern

**Exit criterion:** end-to-end demo — submit one ticket per channel (including at least one non-English), watch it land in the ticketing queue with correct urgency, and see it reflected on the dashboard.

## Phase 8 — Hardening (ongoing)

- [ ] Load test with burst volume (simulate an incident causing a spike in voicemails/screenshots)
- [ ] Confirm Cloud Run concurrency/autoscaling settings handle burst without excessive cold-start latency
- [ ] Add structured logging (`ticket_id` as a common log field) across all services for traceability
- [ ] Set up alerting on: API error rate per stage, dead-letter queue depth, average end-to-end latency

## Timeline Summary

| Phase | Duration | Depends on |
|---|---|---|
| 0. Setup | 0.5 day | — |
| 1. Prove trick points | 1 day | 0 |
| 2. Channel normalizers | 2-3 days | 0, 1 |
| 3. Translation | 1 day | 2 |
| 4. NLU | 1 day | 3 |
| 5. Urgency + storage | 1-2 days | 4 |
| 6. Orchestration | 1-2 days | 2-5 |
| 7. Routing + dashboards | 2-3 days | 5, 6 |
| 8. Hardening | ongoing | 7 |

**Total to a working end-to-end demo: roughly 8-11 working days** for a single engineer; parallelizable across Phase 2's three normalizers if more than one engineer is available.
