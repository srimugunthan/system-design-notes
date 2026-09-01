# System Design: Agentic Employee Onboarding Automation

**Status:** Draft v1
**Owner:** TBD
**Scope:** Multi-agent system to automate new-hire onboarding — account creation, welcome communications, policy summarization, and training scheduling.

---

## 1. Problem Statement & Goals

New-hire onboarding today requires manual coordination across IT (account provisioning), HR (welcome comms, policy walkthroughs), and L&D (training scheduling). This introduces delay, inconsistency, and compliance risk (e.g., policy acknowledgment gaps, access provisioned with wrong scope).

**Goals**
- Reduce onboarding task turnaround from days to hours.
- Guarantee auditability of every action taken on a new hire's behalf.
- Keep humans in the loop for irreversible or high-privilege actions.
- Produce policy summaries that are accurate and traceable to source documents (no hallucinated obligations).

**Non-goals**
- This system does not make hire/no-hire or compensation decisions.
- It does not replace the HRIS/IdP as system of record — it orchestrates against them.

---

## 2. High-Level Architecture

The system is a **supervisor-worker multi-agent architecture**. A single Orchestrator Agent owns the onboarding workflow state machine and delegates discrete, bounded tasks to specialist agents. Agents never call each other directly — all communication passes through the Orchestrator via a structured message bus, which gives a single point for policy enforcement, logging, and rollback.

```
                              ┌─────────────────────────┐
                              │      HR Trigger /        │
                              │   New-Hire Record (HRIS)│
                              └─────────────┬────────────┘
                                            │ onboarding.initiated event
                                            ▼
                              ┌─────────────────────────┐
                              │     ORCHESTRATOR AGENT    │
                              │  (workflow state machine,│
                              │   task routing, approval │
                              │   gate enforcement, audit)│
                              └───────┬───────┬───────┬──┘
                    ┌─────────────────┘       │       └─────────────────┐
                    ▼                         ▼                         ▼
        ┌───────────────────┐    ┌───────────────────────┐   ┌────────────────────┐
        │ Account Provision  │    │  Policy Summarization  │   │ Training Scheduler │
        │       Agent        │    │        Agent           │   │       Agent        │
        │ (IdP/AD/SaaS APIs) │    │ (RAG over policy docs) │   │ (LMS/Calendar API) │
        └─────────┬──────────┘    └───────────┬────────────┘   └──────────┬─────────┘
                  │                            │                          │
                  ▼                            ▼                          ▼
        ┌───────────────────┐    ┌───────────────────────┐   ┌────────────────────┐
        │  Identity Systems  │    │   Policy Doc Store /   │   │   LMS / Calendar   │
        │ (Okta/AD/Workday)  │    │   Vector Index (RAG)   │   │      Systems       │
        └────────────────────┘    └────────────────────────┘   └────────────────────┘

                    All agents also publish to:
                    ┌────────────────────────────────────────┐
                    │   Communication Agent (email/Slack)      │
                    │   — invoked by Orchestrator per stage    │
                    └────────────────────────────────────────┘

                    Every action / message also flows to:
                    ┌────────────────────────────────────────┐
                    │        Audit & Observability Log         │
                    │   (append-only, immutable, queryable)    │
                    └────────────────────────────────────────┘
```

---

## 3. Agent Roster & Ownership

Each agent has a narrow, explicit ownership boundary. No agent has write access to a system outside its lane — this is the core safety invariant of the design.

| Agent | Owns | Does NOT own | Tools / APIs |
|---|---|---|---|
| **Orchestrator** | Workflow state, task sequencing, approval gating, retries, escalation | Any direct write to external systems | Internal state store, message bus |
| **Account Provisioning Agent** | Creating accounts, assigning role-based access, deprovisioning on failure/rollback | Email content, training content | IdP (Okta/Azure AD), HRIS read API |
| **Communication Agent** | Composing and sending welcome emails/Slack messages, tone/branding consistency | Deciding *what* accounts or training to mention (pulls this from Orchestrator-supplied facts only) | Email/SMTP provider, Slack API |
| **Policy Summarization Agent** | Retrieving and summarizing relevant policy documents for the new hire's role/geo | Policy authorship, legal interpretation, final compliance sign-off | RAG pipeline over versioned policy doc store |
| **Training Scheduler Agent** | Finding required/recommended courses, booking calendar slots, LMS enrollment | Content of training curriculum, manager approval of time-off conflicts | LMS API, Calendar API |

**Why a supervisor pattern instead of peer-to-peer:** onboarding is a linear, auditable workflow with clear task boundaries and a need for centralized approval gating. A fully decentralized/peer-to-peer agent mesh would make it harder to guarantee ordering (e.g., don't send "your accounts are ready" before accounts actually exist) and harder to reason about blast radius of a single agent's error.

---

## 4. Inter-Agent Communication

### 4.1 Transport & message shape

All communication is **asynchronous, event-based, and mediated by the Orchestrator** over a message bus (e.g., a task queue / pub-sub layer). Agents do not hold references to each other; they only understand the Orchestrator's task contract.

Every message is a structured, schema-validated JSON object — never free-text instructions between agents. This prevents prompt-injection-style drift where one agent's natural-language output is blindly re-interpreted as instructions by another.

```json
{
  "message_id": "uuid",
  "correlation_id": "onboarding-run-uuid",
  "type": "task.request",
  "from": "orchestrator",
  "to": "account_provisioning_agent",
  "task": "create_account",
  "payload": {
    "employee_id": "E10293",
    "role": "Data Scientist",
    "department": "Risk Analytics",
    "requested_access_tier": "standard"
  },
  "constraints": {
    "requires_human_approval": true,
    "max_privilege_tier": "standard"
  },
  "timestamp": "2026-08-01T09:00:00Z"
}
```

Response contract:

```json
{
  "message_id": "uuid",
  "correlation_id": "onboarding-run-uuid",
  "type": "task.result",
  "from": "account_provisioning_agent",
  "to": "orchestrator",
  "status": "success | failure | needs_approval",
  "result": { "account_id": "acct_8891", "access_tier": "standard" },
  "evidence": { "idp_ticket_id": "IDP-4432" },
  "timestamp": "2026-08-01T09:00:14Z"
}
```

### 4.2 Workflow state machine (owned by Orchestrator)

```
PENDING → ACCOUNTS_PROVISIONING → ACCOUNTS_READY
        → POLICY_SUMMARY_GENERATED → POLICY_REVIEW (human-gated)
        → TRAINING_SCHEDULED
        → WELCOME_COMMS_SENT
        → COMPLETE
        (any state) → FAILED → ROLLBACK → CLOSED
```

The Orchestrator advances state only after receiving a validated `task.result` with `status: success` (or an explicit human approval event for gated steps). This guarantees, for example, that the Communication Agent can never reference an account that hasn't actually been created — it only receives facts the Orchestrator has confirmed.

### 4.3 No shared memory between specialist agents

Specialist agents do not share a common scratchpad or memory store with each other. Each receives only the payload the Orchestrator constructs for its specific task. This limits the blast radius of a hallucinated or corrupted output — a bad summary from the Policy Agent cannot leak into the Account Provisioning Agent's context.

---

## 5. Safety Controls

### 5.1 Human-in-the-loop approval gates

Not every step should be autonomous. Gate placement is risk-weighted:

| Step | Autonomy level | Rationale |
|---|---|---|
| Account creation with standard/default access | Auto with post-hoc audit | Low risk, reversible (deprovision) |
| Account creation with elevated/privileged access | **Hard approval gate** | High blast radius if wrong |
| Welcome email send | Auto | Low risk, templated, reversible-ish |
| Policy summary generation | Auto-generate, **human sign-off before it's shown as authoritative** | Summarization risk of omission/distortion on compliance-sensitive content |
| Training scheduling | Auto, with conflict-check fallback to manager | Low risk |
| Any deviation from the standard role template | **Hard approval gate** | Signals an edge case a human should see |

### 5.2 Least privilege & scoped tool access

- Each agent's tool credentials are scoped to only the systems it owns (Section 3). The Account Provisioning Agent's IdP credential, for instance, cannot touch the LMS.
- The Account Provisioning Agent operates against a **pre-approved role→access-tier mapping table**, not free-form reasoning about what access "seems right" for a title. Any request outside the mapping table routes to human approval rather than the agent improvising.

### 5.3 Grounding controls for the Policy Summarization Agent

This agent is the highest hallucination-risk component since it produces compliance-adjacent natural language.

- Summaries are generated strictly via **RAG against a versioned, access-controlled policy document store** — the agent is not permitted to answer from parametric knowledge alone.
- Every claim in a generated summary must carry a citation back to the specific source policy section (document ID + version + section anchor), stored alongside the summary for audit.
- Summaries are labeled clearly as "informational summary — refer to full policy document at [link]" and are never the system of record for compliance acknowledgment; the new hire's actual sign-off is captured against the source document, not the summary.
- A lightweight consistency check compares summary claims against retrieved source spans before the summary is released; low-confidence or unsupported claims trigger human review rather than being silently dropped or guessed.

### 5.4 Idempotency & rollback

- Every write action (account creation, calendar booking, enrollment) is idempotent, keyed by `correlation_id` + task type, so retries after a timeout/crash cannot create duplicate accounts or double-book training.
- The Orchestrator maintains a rollback plan per completed step (e.g., deprovision account, cancel calendar invite, retract enrollment) so a mid-workflow failure can be unwound cleanly rather than left in a half-onboarded state.

### 5.5 Audit & observability

- Every message on the bus (request and result) is written to an append-only audit log, tagged with `correlation_id`, so a full onboarding run is reconstructable end-to-end for compliance review.
- Structured logs (not just free text) allow querying "show me every account created with elevated access in the last 30 days without a matching approval record" — a real compliance query this design must support.

### 5.6 Failure handling

- Each specialist agent has bounded retries (e.g., 3 attempts with backoff) before escalating to the Orchestrator as `status: failure`.
- The Orchestrator escalates unresolved failures to a human queue rather than looping indefinitely or silently skipping a step.
- Partial completion is always visible: the workflow state is queryable at any point, so a stalled onboarding is detectable rather than silently dropped.

### 5.7 Data handling

- PII (new hire personal details) is passed only in the minimum fields each agent's task requires — e.g., the Policy Summarization Agent never needs the new hire's personal contact info, so it isn't included in its payload.
- Payloads and logs containing PII are encrypted at rest and access-scoped to the audit/compliance role, separate from general engineering access to the message bus.

---

## 6. Sequence Example: Standard New Hire

```
HRIS event → Orchestrator: onboarding.initiated (E10293, Data Scientist, Risk Analytics)
Orchestrator → Account Provisioning Agent: create_account (standard tier)
Account Provisioning Agent → Orchestrator: success (acct_8891)
Orchestrator → Policy Summarization Agent: summarize_policies (role=Data Scientist, geo=IN)
Policy Summarization Agent → Orchestrator: draft summary + citations
Orchestrator → Human (HR reviewer): approval_request (policy summary)
Human → Orchestrator: approved
Orchestrator → Training Scheduler Agent: schedule_training (role template)
Training Scheduler Agent → Orchestrator: success (3 sessions booked)
Orchestrator → Communication Agent: send_welcome_email (account facts, training schedule, policy summary link)
Communication Agent → Orchestrator: success (sent)
Orchestrator: state → COMPLETE
```

---

## 7. Open Questions

- Which system is the source of truth for role→access-tier mapping — HRIS, IdP, or a separate policy service the Orchestrator queries?
- SLA for human approval steps — what happens if a policy-summary reviewer doesn't respond within N hours? (Proposed: escalate to backup approver, do not auto-approve.)
- How are exceptions to standard onboarding (contractors, rehires, cross-border transfers) modeled — as workflow variants, or always routed to full manual review?

---

## 8. Suggested Tech Stack (illustrative)

- **Orchestration:** LangGraph or a similar stateful agent-graph framework, backed by a durable workflow engine (e.g., Temporal) for the state machine and retry semantics.
- **Message bus:** Managed pub/sub (SNS/SQS, Pub/Sub, or Kafka) for task/result events.
- **Policy RAG:** Versioned vector store over policy documents, chunked with section-level metadata for citation.
- **Audit log:** Append-only store (e.g., write-once object storage or a dedicated audit DB) separate from application state.

--
--
# Failure handling

## Failure Points in the Onboarding Agent System

Going through each component/boundary in the architecture:

### 1. External dependency failures
- **IdP (Okta/AD)** — timeout, 5xx, rate limiting, or a stale/incorrect role→access-tier mapping causing a bad write
- **Policy vector store / RAG index** — retrieval timeout, empty/low-relevance results, index out of date (policy updated but not re-indexed)
- **LLM provider** (used by Policy Summarization Agent) — timeout, hang, hallucinated/unsupported claim, rate limiting
- **LMS API** — timeout, enrollment conflict (409), course no longer exists
- **Calendar API** — booking conflict, timeout, wrong timezone/location
- **Email/Slack provider** — send failure, bounce, rate limiting

### 2. Inter-agent communication (message bus) failures
- Message bus itself down/unavailable (nothing can be dispatched or results returned)
- `task.request` or `task.result` fails schema validation (version skew between Orchestrator and an agent after a deploy)
- Message delivered twice (at-least-once delivery) — risk of duplicate processing without idempotency
- Message lost entirely (silent drop) — a task dispatched but result never returned, and no timeout catches it

### 3. Orchestrator failures
- Orchestrator process crashes mid-workflow (in-flight state lost if not durably checkpointed)
- State machine gets into an invalid/unexpected transition (e.g., receives a `task.result` for a step it didn't think was in-flight)
- Orchestrator itself becomes a bottleneck/single point of failure if not horizontally scalable

### 4. Individual agent failures
- Agent process crash mid-task (partial work done, no result ever sent back)
- Agent retries internally beyond its bounded limit and never escalates (silent infinite loop)
- Agent produces a result that passes schema validation but is *semantically* wrong (e.g., Account Provisioning "succeeds" but assigns the wrong access tier due to a stale mapping table)

### 5. Human-in-the-loop failures
- Approver never responds (policy summary sign-off, elevated-access approval) — workflow stalls indefinitely if no SLA/escalation exists
- Approver approves/rejects, but that event is lost before reaching the Orchestrator (network blip, UI bug)
- Wrong person approves (authorization/permission gap) — an integrity failure, not just an availability one

### 6. Idempotency/duplication failures
- A retried or replayed task creates a **duplicate side effect**: two accounts, double calendar booking, two welcome emails
- A DLQ replay races with a delayed original response (the original succeeds *after* being marked failed and requeued)

### 7. Data/consistency failures
- New-hire record in HRIS changes mid-workflow (role or department updated after onboarding already started) — downstream agents may act on stale data
- Partial completion left in an ambiguous state (e.g., account created, but workflow marked `FAILED` before that fact was recorded — account now "orphaned" and invisible to rollback)

### 8. Rollback failures
- A rollback action itself fails (e.g., deprovisioning an account times out) — now you have a failed forward path *and* a failed rollback, with no defined next step

### 9. Cross-cutting / systemic failures
- A circuit breaker trips too aggressively (false positive from a brief blip) and unnecessarily stalls otherwise-healthy runs
- DLQ itself grows unmonitored — the "graveyard problem" — failures are captured but nobody looks at them
- Cascading failure: one dependency's slowness (not full failure) causes timeouts to stack across dependent steps, backing up the whole pipeline

---

This list is the basis for the reliability design — each failure point above maps to a specific pattern (timeout, retry, circuit breaker, idempotency key, fallback, or DLQ routing) in the design I gave earlier. Want me to build a **failure-point → mitigation mapping table** next, so each one traces explicitly to the pattern that covers it?

--
# Reliability design

## Failure Point → Reliability Handling

### 1. External Dependency Failures

| Failure | How to handle reliably |
|---|---|
| IdP timeout/5xx | Bounded retry (3 attempts, exponential backoff) on retryable errors only; per-dependency circuit breaker trips on sustained error rate |
| IdP rate limiting (429) | Retry honoring `Retry-After`; backoff, don't hammer |
| Stale/incorrect role→access-tier mapping | Not retried — route to human approval; fix belongs in the mapping table, not in retry logic |
| Vector store retrieval timeout/empty results | Timeout with tight budget (~2s); on failure, fall back to last-approved summary version if available, else pause and alert |
| LLM hang/timeout | Timeout (~15–20s), 2 retries max; if grounding/citation check still fails, **no further retry** — route to human review (it's a quality failure, not transient) |
| LMS enrollment conflict (409) | Not retryable — falls back to manager-driven manual scheduling, workflow proceeds non-blocked |
| Calendar booking conflict | Same as above — fallback path, not retry |
| Email/Slack send failure | Max **1** automatic retry (avoid duplicate notification risk), then escalate to "send manually" queue |

### 2. Message Bus Failures

| Failure | How to handle reliably |
|---|---|
| Message bus unavailable | Orchestrator buffers/queues locally and replays on reconnect; alert if buffer exceeds threshold |
| Schema validation failure on request/result | Not retried — routed straight to DLQ with `severity: blocking`, flagged for engineering (likely version skew) |
| Duplicate message delivery (at-least-once) | Idempotency key (`correlation_id + task type + entity_id`) makes duplicate processing a no-op |
| Message silently lost (dispatched, no result ever returned) | Per-step timeout on the Orchestrator side (not just the agent side) — if no `task.result` within timeout, treat as failure and retry/escalate, don't wait indefinitely |

### 3. Orchestrator Failures

| Failure | How to handle reliably |
|---|---|
| Orchestrator crash mid-workflow | Durable state checkpointing (e.g., Temporal-backed state machine) — resumes from last committed state, not from scratch |
| Invalid/unexpected state transition (result for a step not in-flight) | Reject and log as an anomaly rather than silently applying; alert if this recurs (signals a bug) |
| Orchestrator as bottleneck/SPOF | Horizontally scalable workflow engine with per-run isolation (each `correlation_id` is independently resumable) |

### 4. Individual Agent Failures

| Failure | How to handle reliably |
|---|---|
| Agent crashes mid-task, no result returned | Orchestrator-side timeout catches this the same as a hang — retries or escalates, doesn't wait forever |
| Agent retries internally beyond bounded limit, never escalates | Hard cap enforced (e.g., 3 attempts) with mandatory escalation to Orchestrator as `status: failure` on exhaustion — no silent infinite loop allowed |
| Agent "succeeds" but result is semantically wrong (e.g., wrong access tier) | Schema validation catches structural errors; semantic errors caught by constraining the agent to a pre-approved mapping table (v1 §5.2) rather than free-form reasoning, plus post-hoc audit sampling |

### 5. Human-in-the-Loop Failures

| Failure | How to handle reliably |
|---|---|
| Approver never responds | SLA timer (e.g., 24 business hours) → escalate to backup approver; **never auto-approve** |
| Approval event lost before reaching Orchestrator | Approval action written durably at the point of click (not just emitted as an event) — Orchestrator polls/reads state, not just listens for a possibly-lost message |
| Wrong person approves (authZ gap) | Approval endpoint enforces role-based access at time of action, independent of workflow logic — checked before the approval is even accepted, not after |

### 6. Idempotency/Duplication Failures

| Failure | How to handle reliably |
|---|---|
| Retry/replay creates duplicate side effect (2 accounts, double booking, 2 emails) | Idempotency key per task type, checked by the downstream system itself where possible (provider-level idempotency headers) plus Orchestrator-side dedup |
| DLQ replay races with delayed original response | Idempotency key ensures replay is a no-op if the original ultimately succeeded; Orchestrator reconciles on whichever response — original or replay — arrives, doesn't blindly trust replay as authoritative |

### 7. Data/Consistency Failures

| Failure | How to handle reliably |
|---|---|
| HRIS record changes mid-workflow (role/dept updated) | Snapshot the new-hire record at `onboarding.initiated` time; if a material change is detected before completion, pause and re-confirm rather than silently acting on stale data |
| Partial completion in ambiguous state (account created, then marked FAILED before recording it) | Write the "step succeeded" fact **before** advancing/failing workflow state — result recording and state transition must be atomic, or state transition waits on confirmed write |

### 8. Rollback Failures

| Failure | How to handle reliably |
|---|---|
| Rollback action itself fails (e.g., deprovision times out) | Rollback follows the same retry/circuit-breaker/DLQ pattern as forward actions — it's not exempt; failed rollback routes to DLQ with `severity: blocking` and pages on-call, since it leaves a real orphaned resource |

### 9. Cross-Cutting / Systemic Failures

| Failure | How to handle reliably |
|---|---|
| Circuit breaker trips on a brief blip (false positive) | Threshold requires a minimum sample size (e.g., 10 requests) not just a percentage, to avoid tripping on 1-2 unlucky failures; half-open state allows fast recovery |
| DLQ grows unmonitored ("graveyard") | Depth/age dashboard + auto-escalating alert when items age past their severity SLA (1h for `blocking`, 24h for `degraded`) |
| Cascading slowness across dependent steps | Each step has its own tight timeout budget (9.1) so a slow dependency fails fast rather than holding downstream steps hostage; circuit breakers stop retry amplification |

--

## When Rollback Is Triggered

Rollback applies when a workflow has **already made a real side effect** (account created, calendar booked, enrollment done) and then a **later, unrecoverable failure** means the run cannot reach `COMPLETE` as a coherent whole. The state machine's `FAILED → ROLLBACK → CLOSED` path exists specifically for this — undoing completed steps so the new hire isn't left half-onboarded.

### Instances that trigger rollback

| Instance | Why rollback (not just retry/fallback) |
|---|---|
| **Elevated-access account created, then the human approval for it is later rejected** (approver reviews post-hoc, or a compliance check flags it) | The write already happened before the gate fully closed, or a downstream review reverses the decision — the account must be deprovisioned, not left standing |
| **Account created, but the new hire's offer is rescinded / start date cancelled mid-workflow** (HRIS signals the run is no longer valid) | Continuing forward is pointless and the account is now an unauthorized artifact — deprovision |
| **A later mandatory step permanently fails after DLQ triage and is deemed unrecoverable** (e.g., role turns out to be invalid, department doesn't exist — a data integrity issue, not a transient one) | No forward path exists; whatever was already provisioned for this run must be unwound rather than left orphaned |
| **Workflow instance is a duplicate of an already-completed run** (detected late, e.g., two `onboarding.initiated` events for the same employee_id due to an HRIS glitch) | The second run's completed steps (its own account creation, its own bookings) are the duplicates and must be rolled back, keeping only the first, legitimate run |
| **Manual operator intervention** (on-call determines a run should not proceed after RCA on a DLQ item — e.g., wrong employee entirely, security concern raised) | Human judgment call overriding the automated path; explicit rollback initiated via runbook procedure |
| **Rollback-triggering rejection during the human policy-summary sign-off gate**, if account/training steps already ran ahead of it (only relevant if your workflow allows any parallelism ahead of that gate — worth confirming this doesn't happen given v1's strictly sequential state machine) | Prevents a partially-communicated, non-compliant onboarding from standing |

### Instances that do NOT trigger rollback

| Instance | Why not |
|---|---|
| Training scheduling fails permanently | Non-blocking — falls back to manual scheduling, workflow still reaches `COMPLETE` |
| Welcome email fails to send | Non-blocking — falls back to "send manually" queue |
| A step is retried and eventually succeeds | Nothing to undo — forward progress achieved |
| A step is stuck in DLQ but still potentially recoverable (transient infra issue) | Rollback is premature — replay is the first option, not undo |

### The design principle
Rollback is reserved for **irreversible-if-left-standing side effects on steps that turned out to be invalid or unauthorized** — mainly Account Provisioning, since that's the only step in this system with real security/compliance blast radius if left in place incorrectly (per v1 §3's ownership table and §5.1's approval gating). Training and communication failures degrade gracefully instead; they don't need undoing because leaving them incomplete isn't harmful, just inconvenient.

--
