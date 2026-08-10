
## LLM-Based Classification of Transaction Narratives & Adverse Media for AML/Fraud Triage

**Status:** Draft v1
**Owner:** Srimugunthan
**Last updated:** 2026-07-16

---

## 1. Problem Statement

Compliance and fraud-ops teams receive high volumes of unstructured text — transaction narratives, wire memo fields, adverse-media snippets, case notes — that need to be triaged into risk categories before a human analyst ever looks at them. Today this is done with brittle keyword/regex rules (high false-positive rate, misses paraphrase and novel typologies) or fully manual review (doesn't scale).

**Goal:** Build a prompt-based LLM classification service that triages narratives into `sanctions_risk`, `fraud_indicator`, `benign`, `needs_review`, with calibrated confidence, full audit trail, and human-in-the-loop escalation — without training a custom model.

**Non-goals (v1):** Fine-tuning a dedicated classifier; multi-language support beyond English; real-time (<200ms) transaction blocking (this is a post-hoc triage system, not an inline transaction-blocking control).

---

## 2. Requirements

### Functional
- Accept a narrative (50–2,000 chars typically) and return a category + confidence + rationale.
- Support both synchronous (ad-hoc analyst lookup) and batch (end-of-day file, 10K–500K narratives) modes.
- Every decision must be traceable: input, prompt version, model version, output, timestamp.
- Escalate low-confidence or high-severity categories to a human review queue automatically.

### Non-functional
- **Auditability** is a first-class requirement, not an afterthought — this is model-risk-governed territory (SR 11-7 / equivalent internal model risk framework applies even though there's no "trained model" in the classical sense; prompts + model version are the model artifact).
- **Determinism where possible** — same input should reliably produce the same output for audit defensibility.
- **Latency:** sync path <3s p95; batch path optimized for cost over latency.
- **Cost ceiling:** triage is high-volume, low-value-per-call — cost per classification must stay low, which shapes several tradeoffs below.
- **Availability:** triage queue must never hard-block on LLM API downtime.

---

## 3. High-Level Architecture

```
                          ┌─────────────────────────┐
                          │   Ingestion Layer       │
                          │  (batch file / API /    │
                          │   streaming queue)       │
                          └────────────┬────────────┘
                                       │
                                       ▼
                          ┌─────────────────────────┐
                          │  Pre-filter / Router    │
                          │  - dedup / cache lookup │
                          │  - cheap rule pre-screen│
                          │  - PII redaction        │
                          └────────────┬────────────┘
                                       │
                     ┌─────────────────┴─────────────────┐
                     ▼                                   ▼
          ┌───────────────────┐               ┌───────────────────────┐
          │  Cache hit path    │               │  LLM Classification   │
          │  (return stored    │               │  Service               │
          │   result)          │               │  - prompt registry     │
          └───────────────────┘               │  - primary model call   │
                                                │  - JSON schema validate│
                                                └──────────┬─────────────┘
                                                            │
                                        ┌───────────────────┼───────────────────┐
                                        ▼                   ▼                   ▼
                              ┌─────────────────┐ ┌─────────────────┐ ┌──────────────────┐
                              │ High confidence  │ │ Low confidence /  │ │ Parse/API failure │
                              │ benign           │ │ sanctions_risk    │ │                    │
                              │ → auto-clear     │ │ → verifier model  │ │ → rule-based       │
                              │                  │ │   (2nd pass, LLM  │ │   fallback         │
                              │                  │ │   or human)       │ │                    │
                              └─────────────────┘ └─────────────────┘ └──────────────────┘
                                        │                   │                   │
                                        └───────────────────┼───────────────────┘
                                                            ▼
                                                ┌───────────────────────┐
                                                │  Audit / Logging Store │
                                                │  (input, prompt ver,   │
                                                │   model ver, output,   │
                                                │   confidence, route)   │
                                                └──────────┬─────────────┘
                                                            │
                                                            ▼
                                                ┌───────────────────────┐
                                                │  Human Review Queue    │
                                                │  (case mgmt system)    │
                                                └───────────────────────┘
```

**Supporting components (not on the hot path but essential):**
- **Prompt Registry** — versioned store of system prompts + few-shot sets (git-backed, tagged releases).
- **Evaluation Harness** — golden set + CI gate that runs before any prompt/model change ships.
- **Drift Monitor** — periodic re-sampling of auto-cleared `benign` items for manual spot-check.
- **Cost/Usage Dashboard** — per-model, per-category token spend.

---

## 4. Key Design Questions and Tradeoffs

This is the part that actually matters — the architecture above is a fairly standard shape; the hard decisions are in these choices.

### 4.1 Zero-shot vs. few-shot vs. fine-tuned

| Approach | Pros | Cons |
|---|---|---|
| Zero-shot | Fastest to ship, no example curation, no drift from stale examples | Weakest on category boundary cases; higher variance on ambiguous inputs |
| Few-shot (chosen) | Strong boundary-case performance with hard negatives; easy to update by editing examples, no retraining | Prompt length cost scales with example count; examples can silently go stale as typologies evolve |
| Fine-tuned classifier | Lowest per-call cost and latency at scale; no prompt-injection surface | Needs labeled training data + retraining pipeline; loses the "reason in natural language" flexibility; becomes a second model-risk artifact to govern |

**Decision:** Few-shot now, with fine-tuning revisited only if volume/cost crosses a threshold where the fine-tuning investment pays back (see 4.5). Few-shot lets you ship without a labeled training set, which you don't have yet.

### 4.2 Confidence calibration — self-reported vs. derived

The model can self-report `high/medium/low` confidence in the JSON output, or you can derive a confidence signal independently (e.g., log-prob of the category token, or agreement rate across repeated sampling / multiple prompt variants).

| Approach | Pros | Cons |
|---|---|---|
| Self-reported (chosen for v1) | Zero extra cost, ships fast | LLMs are known to be poorly calibrated when asked to grade their own certainty — a model can say "high confidence" on a genuinely ambiguous case |
| Log-prob based | Cheap, doesn't need extra calls, more statistically grounded | Not exposed uniformly across all model providers/endpoints; needs recalibration per model version |
| Self-consistency (N samples, majority vote) | Much better-calibrated signal; disagreement across samples is itself a useful "needs_review" trigger | N× the cost and latency; only justifiable for the highest-risk categories |

**Decision:** Self-reported confidence for the first pass (cheap, fast), self-consistency sampling (3–5 calls) as the second-pass verifier **only** for items flagged `sanctions_risk` or `low confidence` on the first pass. This concentrates the expensive technique where it earns its cost.

### 4.3 Model tiering — one model vs. cascade

Running your best model on every narrative is the simplest design and the most expensive at volume.

| Approach | Pros | Cons |
|---|---|---|
| Single strong model for everything | Simple, one prompt to maintain, one eval harness | Wasteful — most narratives are unambiguously benign; expensive at 500K/day scale |
| Cascade: cheap/fast model first, escalate to stronger model on low confidence or flagged categories (chosen) | Big cost reduction; strong model only touches the ~5-15% of cases that need it | Two prompts to maintain and evaluate; escalation logic itself becomes a thing that can silently drift or misfire |

**Decision:** Cascade. This is the single highest-leverage cost decision in the whole system, and it's the same pattern you'd use for InstantReply-style latency/cost optimization — cheap model does volume, expensive model does judgment calls.

### 4.4 Determinism vs. natural language flexibility

Compliance auditors will ask "why did the system classify X as benign," and ideally re-running the same input produces the same answer.

- `temperature=0` gets you close to deterministic but **not guaranteed** deterministic across all providers/model versions (batching, hardware non-determinism can still cause drift at the margins).
- **Tradeoff:** you can pin a specific model version (not just "claude-sonnet") for audit stability, but that means you don't automatically benefit from model upgrades — someone has to explicitly re-validate and re-version before moving to a new model snapshot.

**Decision:** Pin exact model version per prompt version in the registry. Model upgrades go through the same eval-gate as prompt changes — never a silent swap.

### 4.5 Prompt-based vs. fine-tuned — the revisit trigger

This deserves its own callout since it's the biggest architectural fork.

**When to revisit fine-tuning:**
- Volume crosses a threshold where cascade-optimized LLM cost still exceeds the amortized cost of a fine-tuning run + hosting.
- You've accumulated enough audit-trail data from the prompt-based system to use as labeled training data (nice property: the prompt-based system **generates its own future training set** via the audit log).
- Category taxonomy has stabilized (fine-tuning locks you into a schema; prompt-based systems let you add a category by editing a prompt, no retraining).

**Decision:** Explicitly out of scope for v1, but the audit log schema is designed so it can be replayed as a labeled dataset later — this is a deliberate design choice, not an afterthought.

### 4.6 Structured output enforcement

LLMs asked to "return JSON" will occasionally return malformed JSON, add preamble text, or wrap in markdown fences.

| Approach | Pros | Cons |
|---|---|---|
| Prompt-only instruction ("respond ONLY with JSON") | Simple | Non-zero failure rate at scale; someone will eventually parse a markdown-fenced response |
| Strict schema/tool-use enforcement (function calling / structured output mode) | Near-zero parse failures, contract enforced by the API | Slightly more setup; ties you to providers that support it |
| Post-hoc regex/fence-stripping + retry | Cheap patch | Band-aid; doesn't fix root cause, adds retry latency |

**Decision:** Use structured output / tool-use enforcement where available, with a regex-strip-and-retry as a last-resort fallback, and a hard fallback to `needs_review` if both fail. Never let a parse failure silently become "benign."

### 4.7 Human-in-the-loop routing threshold

Where you set the confidence threshold for auto-clearing `benign` vs. routing to a human is a business/regulatory decision disguised as an engineering one.

- **Too permissive** (auto-clear everything not explicitly flagged) → regulator asks what your false-negative rate is and you don't have a good answer.
- **Too conservative** (route everything below "very high" confidence to humans) → you've built an expensive LLM system that doesn't reduce analyst workload, defeating the purpose.

**Decision:** This threshold should not be hardcoded by engineering — it should be a tunable parameter owned jointly with compliance, backed by the golden-set precision/recall numbers per category, and revisited on a fixed cadence (e.g., quarterly) or triggered by drift-monitor findings.

### 4.8 Batch vs. streaming ingestion

| Approach | Pros | Cons |
|---|---|---|
| Batch (end-of-day file) | Cheaper (Batch API discounts), simpler ops, natural fit for EOD reconciliation processes | Latency measured in hours, not seconds — not suitable for anything needing same-day escalation |
| Streaming (per-transaction) | Near-real-time triage | Higher cost per call, more complex infra (queue, backpressure handling), overkill if downstream review is already batch-oriented |
| Hybrid (chosen) | Batch for the EOD bulk file; sync API path reserved for analyst ad-hoc lookups | Two code paths to maintain and keep prompt-consistent |

**Decision:** Hybrid — most volume goes through batch, sync path exists only for analyst-initiated queries, and both paths hit the same prompt registry so behavior never diverges.

---

## 5. Data Flow (Batch Path)

1. EOD transaction/narrative file lands in ingestion store.
2. Pre-filter: dedup against cache (identical narrative text — common with template-generated wire memos), PII redaction pass, cheap rule-based pre-screen to skip obviously benign boilerplate (e.g., internal transfers between own accounts).
3. Remaining narratives → cascade classifier (cheap model first).
4. Route based on category + confidence per section 4.2/4.7.
5. Write every decision to the audit store, regardless of route.
6. Human queue populated with escalated items; analyst decisions fed back into the golden set (with appropriate review, not auto-trusted).

---

## 6. Evaluation & Testing Strategy

- **Golden set:** 100–500 hand-labeled examples per category, weighted toward boundary/edge cases (ambiguous jurisdiction mentions, negation patterns like "not a sanctioned entity," partial name matches).
- **Metrics tracked per category, not just aggregate:** precision, recall, and specifically false-negative rate on `sanctions_risk` (the category where a miss is costliest).
- **Gate:** no prompt or model version ships without beating the current production version on the golden set — no regressions tolerated on `sanctions_risk` recall even if aggregate accuracy improves.
- **Shadow deployment:** new prompt/model version runs in parallel on live traffic (read-only, not affecting routing) for a burn-in period before cutover.
- **Adversarial/red-team testing:** given your RedTeamAgentLoop work, this is a natural fit — test prompt-injection resistance (can narrative text override the system prompt?) and evasion patterns (paraphrase attacks that dodge keyword-adjacent detection).

---

## 7. Monitoring & Drift

- **Category distribution drift:** alert if the proportion of `needs_review` or `sanctions_risk` shifts significantly week-over-week (signals either a new typology or a prompt regression).
- **Escalation-to-overturn rate:** track how often human review overturns the LLM's suggested category — rising overturn rate is an early drift signal.
- **Sampled re-review of auto-cleared benign:** fixed weekly sample, manually re-checked, feeds the false-negative estimate for the category that never reaches a human otherwise.
- **Cost/latency dashboards:** per-model spend, cascade escalation rate (a rising escalation rate quietly erodes the cost savings the cascade was built for).

---

## 8. Risk, Compliance & Governance Considerations

- Treat the **prompt + model version pair** as the model artifact for model risk management purposes — it needs the same documentation rigor (intended use, limitations, validation evidence) as a trained classifier would.
- **Explainability:** the `rationale` field is not just UX — it's what an analyst and, potentially, an auditor will read to understand a decision. Keep it short but substantive, not templated filler.
- **Change control:** prompt edits go through the same review/approval path as code changes, with the eval-gate as an automated check, not a substitute for review.
- **Data handling:** transaction narratives may contain PII/PCI-adjacent data — redaction before the LLM call is not optional, and vendor/API data-retention terms need explicit sign-off from compliance/legal.

---

## 9. Open Questions

- What's the acceptable false-negative rate on `sanctions_risk` that compliance will sign off on, and how is that translated into the confidence threshold in 4.7?
- Does the cascade's cheap-model tier need its own separate golden-set validation, or is it sufficient to validate only the end-to-end routed outcome?
- At what volume does fine-tuning's amortized cost beat the cascade (see 4.5) — worth a standalone cost model rather than a guess.
- Should the audit log double as the labeled dataset for a future fine-tuned model, and if so, what review step turns "LLM output + analyst overturn/confirm" into a trustworthy label?
