# Production LLM Systems — Operational Q&A

*Topics: MCP schema versioning, safe rollback, agent evaluation, cost governance, and privacy-safe observability*

---

## 1. How do you version MCP schemas and ensure backward compatibility?

**Treat MCP tool/resource schemas as a public API contract, not an internal implementation detail.**

- **Semantic versioning per tool, not per server.** Each tool definition (name, input schema, output schema) carries its own version (e.g., `get_transaction_history@v2`). A server-wide version bump forces every client to re-validate everything, even tools that didn't change.
- **Additive-first schema evolution.** New optional fields with sensible defaults are safe. Renaming a field, tightening a type, or removing a field is a breaking change and requires a new tool version, not an in-place mutation of the old one.
- **Dual-serve during migration.** Run `tool_v1` and `tool_v2` side by side for a deprecation window (typically 1–2 release cycles). Route by client-declared capability during the handshake, log which version each caller is on, and only retire `v1` once traffic drops to zero or a hard deadline passes.
- **Contract tests in CI.** Golden request/response fixtures per tool version are replayed on every schema change; a failing fixture blocks merge. This catches accidental breaking changes before they reach agents in production.
- **Schema registry with a compatibility checker.** A lightweight registry (even just JSON Schema files in a versioned repo) with a CI check similar to Confluent Schema Registry's compatibility modes (`BACKWARD`, `FORWARD`, `FULL`) — reject any PR that fails backward compatibility unless explicitly overridden with a major version bump.
- **Fail loud, not silent, on mismatch.** If an agent calls a deprecated or unknown tool version, return a structured error the agent/orchestrator can react to (e.g., fall back to `v1`, or surface to a human), rather than letting a malformed call silently degrade output quality.

---

## 2. How do you roll back a broken prompt or agent flow safely?

**The core principle: prompts and agent graphs are deployable artifacts, so they need the same rigor as code — versioning, canarying, and instant revert.**

- **Version every prompt/flow as an immutable artifact.** Store prompts, system messages, and LangGraph/agent-graph definitions in a registry keyed by content hash or semantic version, decoupled from the application code deploy cycle. Never edit a prompt "in place" in production config.
- **Canary before full rollout.** New prompt/flow versions go to a small traffic slice (1–5%) or a specific low-risk tenant segment first, with automated comparison against the previous version's eval scores (see Q3) before promotion.
- **Feature-flag the flow version.** Keep the previous N versions hot and route via a flag/config value, not a redeploy. Rollback becomes a config change (seconds), not a code revert and redeploy (minutes to hours).
- **Automated rollback triggers.** Wire alerting on leading indicators — spike in tool-call errors, output-schema validation failures, refusal rate, latency, or a drop in a cheap proxy eval score — to auto-revert to the last known-good version without waiting for a human to notice.
- **Shadow mode for high-risk changes.** For agent flows touching money movement, compliance decisions, or customer communication, run the new flow in shadow (compute output, don't act on it) against live traffic and diff against the current production flow before ever promoting.
- **Post-rollback root cause before re-attempting.** Capture the triggering traces (see Q5) so the failure mode is understood — a bad few-shot example, a tool schema drift, a context-window truncation — before re-attempting the change, rather than just re-rolling forward blind.

---

## 3. How do you evaluate agent performance vs single-model prompts?

**Agents need evaluation at two levels: end-to-end outcome quality, and the incremental value the agentic scaffolding adds over a single well-prompted call.**

- **Establish a single-prompt baseline first.** Before crediting an agent architecture with a win, run the best achievable single-shot/single-model prompt on the same task and same eval set. If the agent doesn't clear that baseline by a meaningful margin, the added latency, cost, and failure surface aren't justified.
- **Separate task success from process quality.** Track two axes:
  - *Outcome metrics* — task completion rate, correctness against a labeled/rubric-graded gold set, downstream business metric (e.g., false-positive rate on a fraud-flagging agent).
  - *Process metrics* — number of tool calls to completion, retry/error rate, wasted or redundant calls, step count vs. an efficient reference trajectory.
- **Use trajectory-level evaluation, not just final-answer grading.** For multi-step agents, grade intermediate steps (did it call the right tool, in the right order, with valid arguments) in addition to the final output — a correct final answer reached via a flaky or unsafe path is a latent risk, not a win.
- **LLM-as-judge with calibration.** For open-ended outputs, use a stronger model as judge against a rubric, but validate the judge against a human-labeled subset periodically to catch judge drift or bias — don't treat judge scores as ground truth without spot-checking.
- **Cost- and latency-normalized comparison.** Report quality *per dollar* and *per second*, not quality in isolation. An agent that's 3 points better but 8x the cost and 5x the latency is rarely the right production trade-off; make that trade-off explicit to stakeholders rather than implicit in a leaderboard number.
- **Regression suite, not one-off benchmark.** Maintain a fixed eval set (with adversarial/edge cases) that reruns on every prompt or flow change, so "agent vs. single-prompt" isn't a one-time decision but a continuously monitored one as both evolve.

---

## 4. How do you cap cost per tenant when using agentic workflows?

**Agentic workflows have unbounded worst-case cost (loops, retries, runaway tool chains), so cost control needs both hard ceilings and graceful degradation.**

- **Hard per-request and per-session budgets.** Set a token/dollar ceiling per agent run (e.g., max total tokens across all steps, max tool calls) enforced by the orchestrator itself — not just monitored after the fact. When the budget is hit, terminate gracefully with a partial result or a "need more budget to continue" signal rather than silently cutting off mid-task.
- **Per-tenant quota with tiered limits.** Track spend at the tenant level (daily/monthly rolling window) and enforce quotas proportional to their plan tier. Use a token bucket or leaky bucket algorithm so bursts are allowed but sustained overuse is throttled.
- **Step-count and depth limits on the agent graph.** Cap max iterations of any loop (ReAct-style reasoning loops, retry loops, multi-agent handoffs) independent of token cost — a cheap infinite loop of small calls can still be an operational and latency problem even if individual tokens are inexpensive.
- **Model routing by task complexity.** Don't run every step on the most expensive model. Route simple sub-tasks (classification, extraction, tool-argument formatting) to a cheaper/smaller model and reserve the frontier model for the steps that actually need its reasoning — this is often the single biggest cost lever in an agentic pipeline.
- **Cache aggressively.** Cache tool results, retrieval results, and even full agent responses for repeated/similar queries (semantic cache with a similarity threshold) so cost doesn't scale linearly with redundant traffic from the same tenant.
- **Real-time cost telemetry with circuit breakers.** Emit cost-per-call metrics tagged by tenant, agent flow version, and model, and wire a circuit breaker that pauses a tenant's agentic workflows (falling back to a cheaper deterministic path or a queued/manual-review state) if their spend rate spikes anomalously — this catches both legitimate usage surges and bugs like infinite retry storms.
- **Pre-flight cost estimation for expensive flows.** For flows with a wide cost variance (e.g., open-ended research agents), estimate expected cost before execution based on historical distribution and either warn, cap, or require explicit approval above a threshold.

---

## 5. How do you log prompts and traces without leaking PII?

**Observability and privacy are usually framed as a trade-off, but the right pattern is to make redaction part of the trace pipeline itself, so full debugging fidelity and PII safety aren't mutually exclusive.**

- **Redact before storage, not after.** Run inputs and outputs through a PII detection/redaction layer (regex + NER-based detector for names, account numbers, SSN/Aadhaar-equivalents, emails, phone numbers) as part of the logging middleware itself, before anything touches a persistent store — never log raw first, "clean up later."
- **Tokenize, don't just mask.** Replace detected PII with reversible tokens (`{{ACCOUNT_NUMBER_1}}`) mapped in a separate, tightly access-controlled vault, rather than irreversible masking. This preserves the ability to debug a trace structurally (same entity referenced consistently across steps) without ever storing the raw value in the main trace store.
- **Structural trace logging over raw-text logging.** Log the *shape* of the interaction — tool calls made, arguments' field names (not values, or redacted values), token counts, latencies, model versions, step sequence — as structured fields. This gets you most debugging value even in a partial-redaction failure mode, since a redaction miss on structured metadata is far less damaging than a miss in free text.
- **Field-level classification at the schema boundary.** For MCP tool calls specifically, tag each field in the tool's input/output schema as PII / non-PII / sensitive-but-non-PII at definition time, so the logging layer knows what to redact without needing to run detection on every field of every call.
- **Sampling with escalation, not blanket full-fidelity logging.** Log full (redacted) traces for a sampled percentage of traffic for quality monitoring, but log only structural metadata for the rest; escalate to full trace capture automatically when an error, low-confidence, or anomaly signal fires on a given request.
- **Separate retention policies for trace tiers.** Redacted structural traces can be retained long-term for eval/regression use; anything containing tokenized-but-reversible PII gets a much shorter retention window and stricter access control, aligned with data-retention regulations relevant to financial services (e.g., RBI data localization / retention norms).
- **Audit the redaction layer itself.** Periodically run known-PII test inputs through the full pipeline and verify they're caught — treat the redaction detector as a model with its own precision/recall requirements and its own eval set, since a silent regression here is a compliance incident, not just a bug.

---

*Document generated for internal reference — production LLM system operations (MCP tooling, agentic workflows, financial services context).*
