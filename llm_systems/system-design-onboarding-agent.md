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
