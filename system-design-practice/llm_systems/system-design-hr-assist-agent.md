# HR AI Assistant — System Design & Implementation Plan

> **Scope:** An AI assistant embedded inside an HR SaaS platform supporting policy Q&A, document generation (offer letters, policy documents), and sensitive-case escalation — designed for security, cost-control, and full observability.

---

## Table of Contents

1. [High-Level Architecture](#1-high-level-architecture)
2. [Component Specifications](#2-component-specifications)
3. [Context Engineering Policy](#3-context-engineering-policy)
4. [Model Selection Strategy](#4-model-selection-strategy)
5. [Security & Compliance](#5-security--compliance)
6. [Observability & Evaluation](#6-observability--evaluation)
7. [Failure Modes & Fallbacks](#7-failure-modes--fallbacks)
8. [Cost Control Architecture](#8-cost-control-architecture)
9. [Data Flow Diagrams](#9-data-flow-diagrams)
10. [Implementation Roadmap](#10-implementation-roadmap)

---

## 1. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          HR SaaS Platform                                     │
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                         UI Layer                                         │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │  │
│  │  │ Chat Widget  │  │  Doc Editor  │  │  Admin Panel │  │  Escalation│  │  │
│  │  │  (React/Vue) │  │  (Rich Text) │  │  (RBAC Mgmt) │  │   Inbox    │  │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘  │  │
│  └─────────┼─────────────────┼─────────────────┼────────────────┼──────────┘  │
│            │                 │                 │                │              │
│  ┌─────────▼─────────────────▼─────────────────▼────────────────▼──────────┐  │
│  │                     API Gateway (Kong / AWS API GW)                       │  │
│  │   AuthN/AuthZ · Rate Limiting · TLS Termination · Request Tracing         │  │
│  └─────────────────────────────────┬─────────────────────────────────────────┘  │
│                                    │                                             │
│  ┌─────────────────────────────────▼─────────────────────────────────────────┐  │
│  │                    AI Orchestrator (LangGraph / Custom)                    │  │
│  │                                                                             │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐  ┌────────────────┐  │  │
│  │  │  Intent     │  │  Context     │  │  Router /   │  │  Guard Rails   │  │  │
│  │  │  Classifier │  │  Builder     │  │  Dispatcher │  │  (PII / Tone)  │  │  │
│  │  └──────┬──────┘  └──────┬───────┘  └──────┬──────┘  └───────┬────────┘  │  │
│  └─────────┼────────────────┼─────────────────┼──────────────────┼───────────┘  │
│            │                │                 │                  │               │
│  ┌─────────▼────────┐  ┌────▼──────────────┐  │           ┌──────▼───────────┐  │
│  │ Retrieval Service│  │  Prompt Template  │  │           │  Escalation      │  │
│  │                  │  │  Engine           │  │           │  Handler         │  │
│  │ ┌──────────────┐ │  │ (versioned)       │  │           │  (HRBP routing)  │  │
│  │ │  Embedding   │ │  └───────────────────┘  │           └──────────────────┘  │
│  │ │  Service     │ │                         │                                  │
│  │ └──────┬───────┘ │  ┌──────────────────────▼──────────────────────────────┐  │
│  │        │         │  │                   Model Layer                         │  │
│  │ ┌──────▼───────┐ │  │                                                       │  │
│  │ │  Vector DB   │ │  │  ┌────────────────┐   ┌───────────────────────────┐  │  │
│  │ │  (Pinecone / │ │  │  │ Internal Models│   │  External Providers       │  │  │
│  │ │  Weaviate)   │ │  │  │ Fine-tuned SLM │   │  Claude / GPT-4o / Gemini │  │  │
│  │ └──────────────┘ │  │  │ (Phi-3 / Llama)│   │  (via LiteLLM proxy)      │  │  │
│  └──────────────────┘  │  └────────────────┘   └───────────────────────────┘  │  │
│                        └──────────────────────────────────────────────────────┘  │
│                                                                                   │
│  ┌──────────────────────┐  ┌─────────────────────┐  ┌──────────────────────┐    │
│  │  Relational Store    │  │  Audit & Logging     │  │  Monitoring & Alerts │    │
│  │  (PostgreSQL + RLS)  │  │  (Kafka + S3 + SIEM) │  │  (Grafana / Datadog) │    │
│  └──────────────────────┘  └─────────────────────┘  └──────────────────────┘    │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Component Specifications

### 2.1 UI Layer

| Component | Technology | Responsibility |
|---|---|---|
| Chat Widget | React + WebSocket | Real-time streaming Q&A, citation display |
| Doc Editor | ProseMirror / TipTap | Draft review, inline AI suggestions, tracked changes |
| Admin Panel | React Admin | RBAC config, model version selection, prompt overrides |
| Escalation Inbox | React + SSE | HR BP queue, case history, resolution tracking |

**Key design decisions:**
- Chat is streaming (SSE/WebSocket) to reduce perceived latency
- Document editor shows diff view when AI modifies a template
- All actions disabled until explicit user consent event is captured in session

---

### 2.2 API Gateway

**Responsibilities:**
- JWT / OAuth2 validation (SSO-integrated via SAML/OIDC)
- Per-tenant rate limiting: `X requests/min` configurable in control plane
- Request tracing: inject `x-trace-id`, `x-tenant-id`, `x-user-role` headers
- IP allowlisting for enterprise tenants
- TLS 1.3 termination; internal mTLS to orchestrator

**Rate limiting tiers:**

| Tier | Q&A (RPM) | Doc Gen (RPD) | Cost Cap ($/day) |
|---|---|---|---|
| Free | 20 | 5 | $0.50 |
| Professional | 200 | 50 | $10 |
| Enterprise | 2000 | 500 | Negotiated |

---

### 2.3 AI Orchestrator

The orchestrator is stateful and implemented as a **LangGraph**-style DAG with explicit node transitions. It handles three primary workflows:

**Workflow A — Policy Q&A:**
```
[Intent Classify] → [PII Redact] → [Context Build] → [Retrieve] → [Generate] → [Guardrails] → [Response]
```

**Workflow B — Document Generation:**
```
[Intent Classify] → [Template Select] → [Variable Extract] → [PII Redact]
→ [Draft Generate] → [Compliance Check] → [Human Review Gate?] → [Return Draft]
```

**Workflow C — Sensitive Case Escalation:**
```
[Intent Classify] → [Sensitivity Score] → [PII Redact] → [Summarize]
→ [Route to HRBP] → [Notification] → [Audit Log]
```

**Intent Classifier:**
- Fast, cheap model (Claude Haiku / Phi-3 Mini) — ~$0.0001/request
- Outputs: `{intent: "qa"|"doc_gen"|"escalation"|"ambiguous", confidence: 0.0-1.0, sensitivity: "low"|"medium"|"high"}`
- Confidence < 0.7 triggers clarification prompt; sensitivity "high" forces escalation path

---

### 2.4 Retrieval Service

**Architecture: Hybrid RAG**

```
Query
  │
  ├── Dense Retrieval (vector similarity, top-k=10)
  │     └── Embedding model: text-embedding-3-small or E5-large-v2 (self-hosted)
  │
  ├── Sparse Retrieval (BM25 keyword, top-k=10)
  │     └── Elasticsearch / OpenSearch
  │
  └── Reciprocal Rank Fusion → re-rank top-5 → return with metadata
```

**Document indexing pipeline:**

```
Raw Doc (PDF/DOCX/HTML)
  → OCR if scanned
  → Chunking (512 tokens, 50-token overlap, sentence-boundary aware)
  → Metadata extraction: {doc_type, version, effective_date, jurisdiction, tenant_id}
  → Embedding generation (batched, async)
  → Vector DB upsert with namespace = tenant_id
  → BM25 index update
  → Version tag on existing chunks (soft delete old version)
```

**Retrieval metadata contract:**

```json
{
  "chunk_id": "uuid",
  "doc_id": "uuid",
  "tenant_id": "tenant-123",
  "doc_type": "leave_policy|code_of_conduct|offer_template",
  "version": "2024-Q4",
  "effective_date": "2024-10-01",
  "jurisdiction": "IND|USA|EU",
  "sensitivity": "public|internal|confidential",
  "text": "...",
  "score": 0.87
}
```

---

### 2.5 Model Layer

See Section 4 for full model selection rationale.

**LiteLLM proxy** sits between orchestrator and all model providers:
- Unified API surface (OpenAI-compatible)
- Per-model cost tracking with budget enforcement
- Automatic fallback routing if primary model returns 5xx
- Request/response logging to audit store (stripped of PII before log write)

---

### 2.6 Vector DB

| Dimension | Choice | Rationale |
|---|---|---|
| Primary | Pinecone (managed) | Serverless, per-namespace tenant isolation, no ops overhead |
| Self-hosted alt | Weaviate / Qdrant | For data residency requirements (EU/India) |
| Namespace strategy | 1 namespace per tenant | Hard isolation, prevents cross-tenant retrieval |
| Index type | HNSW | Best recall/latency tradeoff for <10M vectors |
| Dimensions | 1536 (OpenAI) or 1024 (E5-large) | |

**Vector DB SLA target:** p99 query latency < 50ms

---

### 2.7 Relational Store (PostgreSQL)

**Core tables:**

```sql
-- Tenant configuration
tenants (id, name, plan, cost_cap_daily, model_config_json, created_at)

-- User & RBAC
users (id, tenant_id, email, role ENUM('employee','hr_bp','hr_admin','super_admin'))
permissions (role, resource, action)

-- Conversation history
conversations (id, tenant_id, user_id, created_at, sensitivity_level, status)
messages (id, conv_id, role, content_encrypted, tokens_used, model_id, latency_ms, cost_usd, created_at)

-- Document generation
generated_docs (id, tenant_id, user_id, doc_type, template_version, status, content_encrypted, created_at)

-- Escalations
escalation_cases (id, conv_id, assignee_id, severity, summary_encrypted, status, created_at, resolved_at)

-- Audit log (append-only)
audit_events (id, tenant_id, user_id, action, resource_type, resource_id, ip_address, trace_id, timestamp)

-- Prompt versions
prompt_versions (id, workflow, version, content_hash, content, is_active, created_by, created_at)
```

**Row-Level Security:** All multi-tenant tables have RLS policies enforcing `tenant_id = current_setting('app.tenant_id')`.

---

### 2.8 Audit & Logging

**Three-tier logging architecture:**

```
Application Events
       │
       ├── Structured JSON → Kafka topic: hr-ai-audit
       │                          │
       │                    ┌─────▼──────┐
       │                    │ Stream     │
       │                    │ Processor  │  ← PII scrubber runs here
       │                    └─────┬──────┘
       │                          │
       │               ┌──────────┴──────────┐
       │               │                     │
       │          ┌────▼────┐          ┌─────▼─────┐
       │          │ S3 Cold │          │  SIEM     │
       │          │ Archive │          │ (Splunk/  │
       │          │ (7 yr)  │          │  Elastic) │
       │          └─────────┘          └───────────┘
       │
       └── Real-time → Application DB (90 days hot)
```

**Mandatory audit fields per event:**
```json
{
  "trace_id": "uuid",
  "tenant_id": "string",
  "user_id": "string",
  "user_role": "string",
  "action": "qa_query|doc_generate|escalate|admin_config_change",
  "model_id": "claude-sonnet-4|gpt-4o|...",
  "prompt_version": "v2.3.1",
  "tokens_input": 1200,
  "tokens_output": 450,
  "cost_usd": 0.0042,
  "latency_ms": 1840,
  "retrieval_chunks_used": 3,
  "pii_detected": false,
  "escalated": false,
  "timestamp": "ISO8601"
}
```

---

### 2.9 Monitoring & Alerts

**Dashboard layers:**

| Layer | Tool | Key Panels |
|---|---|---|
| Infrastructure | Grafana + Prometheus | CPU/memory, DB connections, Kafka lag |
| AI Operations | Custom Grafana + LangSmith | Latency P50/P95/P99, token usage, cost burn rate |
| Quality | Evidently AI / Arize | Hallucination rate, answer relevance, retrieval recall |
| Business | Metabase | Daily active users, doc gen volume, escalation rate |

**Alert thresholds:**

| Metric | Warning | Critical |
|---|---|---|
| P99 latency (Q&A) | > 3s | > 8s |
| P99 latency (Doc Gen) | > 10s | > 30s |
| Daily cost burn | > 80% cap | > 95% cap |
| Hallucination rate (online) | > 5% | > 15% |
| Retrieval recall@5 (offline) | < 0.75 | < 0.60 |
| Error rate (5xx) | > 1% | > 5% |

---

## 3. Context Engineering Policy

### 3.1 Context Window Composition

For every request to the LLM, the context is assembled deterministically by the **Context Builder** component. The composition policy is version-controlled alongside prompt templates.

**Context layers (in order, from highest to lowest priority):**

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1: System Prompt (versioned, immutable per request)       │
│  - Role definition, tone, safety instructions                    │
│  - Tenant-specific overrides (tone, jurisdiction)                │
│  - Hard constraints: "Never give legal advice", "Always cite"    │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 2: Tenant Context (semi-static, cached 1hr)               │
│  - Company name, industry                                        │
│  - Active policy versions in effect                              │
│  - Jurisdiction (IND / USA / EU — affects policy selection)      │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 3: User Context (per-request, from DB)                    │
│  - User role (employee/HRBP/admin) — controls what's returned   │
│  - Department — for role-specific policy retrieval               │
│  - Employment type (FTE/contractor) — for offer letter scope     │
│  ⚠️ REDACTED: Name, email, salary, performance rating, health    │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 4: Retrieved Chunks (dynamic, from Retrieval Service)     │
│  - Max 5 chunks, each tagged with source + version               │
│  - Ordered by reranker score                                     │
│  - Total retrieved context budget: 3,000 tokens                  │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 5: Conversation History (sliding window)                  │
│  - Last 6 turns (3 user + 3 assistant)                           │
│  - Summarized if > 2,000 tokens                                  │
│  ⚠️ REDACTED before storage: PII, salary figures, health info    │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 6: Current User Message                                   │
│  - PII stripped by regex + NER model before sending to LLM      │
│  - Original stored encrypted in DB                               │
└─────────────────────────────────────────────────────────────────┘
```

**Total context budget allocation:**

| Layer | Token Budget |
|---|---|
| System prompt | 500 |
| Tenant context | 200 |
| User context | 100 |
| Retrieved chunks | 3,000 |
| Conversation history | 2,000 |
| Current message | 800 |
| **Total input budget** | **6,600** |
| **Output budget (Q&A)** | **1,000** |
| **Output budget (Doc Gen)** | **4,000** |

---

### 3.2 PII Redaction Policy

**Automatic redaction before any LLM call:**

| PII Category | Detection Method | Action |
|---|---|---|
| Email addresses | Regex | Replace with `[EMAIL]` |
| Phone numbers | Regex | Replace with `[PHONE]` |
| Person names | NER (spaCy / Presidio) | Replace with `[NAME]` |
| SSN / Aadhaar / PAN | Regex | Replace with `[GOVT_ID]` |
| Salary / compensation | Regex + NER | Replace with `[SALARY]` |
| Bank account numbers | Regex | Replace with `[BANK_ACCT]` |
| Medical / health info | NER + keyword list | Replace with `[HEALTH_INFO]` |
| Performance ratings | Keyword heuristic | Replace with `[PERF_DATA]` |

**Redaction is applied at two points:**
1. Inbound (user query → orchestrator): before building LLM context
2. Outbound (LLM response → audit store): before persisting to logs

**Redaction is NOT applied to:**
- Template variable slots in doc generation (these are filled server-side, never sent to LLM raw)
- Escalation summaries sent to HR BPs (these are authorized to see PII)

---

### 3.3 Context Versioning

```
context_policy/
  v1.0.0/
    system_prompt.txt          ← Base system prompt
    tenant_context_schema.json ← What tenant fields to include
    user_context_schema.json   ← What user fields to include
    redaction_rules.json       ← PII patterns + actions
    chunk_budget.json          ← Token allocations
  v1.1.0/
    ...                        ← Incremental changes
  CHANGELOG.md
```

**Versioning rules:**
- Every request logs `context_policy_version` in the audit record
- Versions are semantic: `MAJOR.MINOR.PATCH`
  - MAJOR: breaking change in context structure (requires A/B eval before rollout)
  - MINOR: new context field added (backward-compatible)
  - PATCH: wording tweak, does not affect retrieval
- Rollback: set `is_active=false` on prompt_versions row; orchestrator reads active version at startup with 5-min cache TTL

---

## 4. Model Selection Strategy

### 4.1 Task-to-Model Matrix

| Task | Primary Model | Fallback | Rationale |
|---|---|---|---|
| Intent classification | Claude Haiku / Phi-3 Mini | Rule-based classifier | Low latency, cheap, structured output |
| Policy Q&A (simple) | Claude Sonnet (via RAG) | GPT-4o mini | Strong instruction following, cites sources |
| Policy Q&A (complex / multi-doc) | Claude Sonnet / GPT-4o | Claude Haiku + reask | Longer context, better reasoning |
| Offer letter generation | Claude Sonnet | GPT-4o | Instruction-following, format adherence |
| Legal/compliance policies | Claude Sonnet + compliance guardrails | Human escalation | Risk surface too high for smaller models |
| Escalation summarization | Claude Haiku | Rule-based template | Fast triage, not customer-facing |
| Embedding | text-embedding-3-small | E5-large-v2 (self-hosted) | Cost vs. data-residency tradeoff |
| PII detection (NER) | Presidio (local) | spaCy en_core_web_lg | Never send PII to external API for detection |

---

### 4.2 Fine-Tune vs Prompt-Tune vs RAG Decision Framework

```
Is the knowledge static and retrievable from documents?
       │
       YES → Use RAG first. Fine-tuning is not needed.
       │
       NO → Is it a style/format/behavior issue?
               │
               YES → Prompt engineering or few-shot examples first.
               │     If still failing after 20+ examples → consider fine-tune.
               │
               NO → Is it domain-specific terminology / entity recognition?
                       │
                       YES → Fine-tune a small model (Phi-3 / Llama-3.1-8B) on HR corpus
                       │     Use LoRA/QLoRA for cost efficiency
                       │
                       NO → Is it a task the model cannot do via prompting?
                               │
                               YES → Fine-tune on labeled task examples
                               NO → Escalate to human or use larger model
```

**Concrete recommendations for HR SaaS:**

| Use Case | Approach | Why NOT fine-tune |
|---|---|---|
| Policy Q&A | RAG (hybrid) | Policy content changes quarterly; fine-tuning would go stale |
| Offer letter gen | Prompt + template variables | Format is fixed; variability is in data, not reasoning |
| Sensitivity classification | Fine-tuned small model | Proprietary HR taxonomy not in base model training |
| Jurisdiction routing | Prompt + rules engine | Logic is explicit; LLM adds no value over if/else |
| Tone enforcement | System prompt + sampling params | Prompt-engineering sufficient |

---

### 4.3 Model Versioning & Promotion

```
Model versions pinned in config store:
  qa_model: "claude-sonnet-4-20251001"
  docgen_model: "claude-sonnet-4-20251001"
  intent_model: "claude-haiku-4-5-20251001"

Promotion gates:
  1. Offline eval: golden dataset score must improve or not regress > 2%
  2. Shadow mode: new model runs in parallel for 48hrs, logs compared
  3. Canary: 5% traffic for 24hrs, SLO compliance checked
  4. Full rollout or rollback
```

---

## 5. Security & Compliance

### 5.1 PII Handling Architecture

```
User Input
    │
    ▼
[PII Detector: Presidio + custom HR rules]
    │
    ├── PII found → Tokenize (replace with reversible token stored in Redis 1hr TTL)
    │              → Log: "PII detected, redacted before model call"
    │
    └── Clean → Pass to Context Builder
                      │
                      ▼
              [LLM Call - no PII in prompt]
                      │
                      ▼
              [Response — scan for PII bleed-through]
                      │
                      ▼
              [De-tokenize for authorized user display]
                      │
                      ▼
              [Store encrypted in DB, PII stripped from audit logs]
```

**Data flows that MUST never contain PII:**
- Vector DB chunk contents (anonymized at index time)
- Audit logs sent to SIEM
- LLM provider API calls
- Monitoring dashboards / metrics labels

---

### 5.2 Encryption

| Data State | Standard | Key Management |
|---|---|---|
| At rest (DB) | AES-256 (column-level for PII fields) | AWS KMS / HashiCorp Vault, tenant-specific DEKs |
| At rest (S3 audit) | SSE-S3 with KMS | Per-tenant key rotation every 90 days |
| In transit | TLS 1.3 | Certificate Pinning for mobile clients |
| Redis tokens | TTL-bounded, AES-128 | Ephemeral, not persisted |
| Vector embeddings | AES-256 namespace-level encryption | KMS-backed |

---

### 5.3 RBAC Model

```
ROLE: super_admin
  - Can access all tenants, all data
  - Can change model config, prompt versions
  - Requires MFA + reason logging

ROLE: hr_admin
  - Full access within tenant
  - Can view all conversations (including PII)
  - Can configure escalation routing

ROLE: hr_bp (HR Business Partner)
  - Access to escalated cases assigned to them
  - Can view full conversation with PII
  - Cannot change config

ROLE: employee
  - Can only see own conversations
  - Q&A and doc gen enabled
  - Cannot see other employees' data

Enforcement points:
  1. API Gateway: role extracted from JWT, injected as header
  2. Orchestrator: role governs context building (what user_context to include)
  3. DB: PostgreSQL RLS (tenant_id + user_id checks)
  4. Vector DB: namespace-scoped queries (tenant_id enforced server-side)
```

---

### 5.4 Consent & Data Retention

**Consent capture:**
- First use: explicit consent modal (stored with timestamp, IP, version of privacy policy)
- Consent version tracked; re-consent triggered on policy updates
- Right-to-erasure workflow: cascade delete across conversations, messages, vectors, audit logs (except immutable SIEM records which are anonymized)

**Data retention matrix:**

| Data Type | Hot Storage | Cold Archive | Deletion Trigger |
|---|---|---|---|
| Conversation messages | 90 days | 2 years | User request or tenant offboarding |
| Generated documents | 1 year | 7 years (legal hold) | Per jurisdiction rules |
| Audit logs (SIEM) | 30 days | 7 years | Non-deletable (regulatory) |
| Embeddings / vectors | Active subscription | Purged on offboarding | Tenant deletion |
| PII tokens (Redis) | 1 hour | None | Auto-expiry |
| Model call logs | 30 days | 90 days | Rolling window |

---

### 5.5 External Model Provider Controls

When routing to Claude / GPT-4o / Gemini:
- **Zero Data Retention (ZDR) agreements** required with all providers for enterprise tenants
- **No training opt-in**: API calls use `training=false` flag where available
- **Data residency**: route to provider endpoints matching tenant jurisdiction (EU → EU endpoints)
- **Payload inspection**: LiteLLM proxy logs input/output tokens, enforces max token limits, rejects requests exceeding risk threshold

---

## 6. Observability & Evaluation

### 6.1 Offline Evaluation (Golden Dataset)

**Golden dataset composition for HR domain:**

| Subset | Size | Type | Refresh Cadence |
|---|---|---|---|
| Policy Q&A | 200 Q&A pairs | Reference answers from HR team | Quarterly |
| Edge cases | 50 adversarial | Trick questions, out-of-scope | Semi-annual |
| PII handling | 100 queries | Contains PII, expected redaction | Monthly |
| Doc generation | 30 templates | Human-approved gold documents | Per template update |
| Escalation trigger | 50 scenarios | Expected escalation=true/false | Quarterly |

**Evaluation metrics:**

```
Q&A Metrics:
  - Faithfulness: Is the answer grounded in retrieved chunks? (LLM-as-judge)
  - Answer relevance: Does it address the question? (embedding similarity)
  - Citation accuracy: Are cited chunks actually relevant? (manual spot-check)
  - Hallucination rate: Claimed facts not in retrieved context (LLM-as-judge)

Doc Generation Metrics:
  - Template variable fill rate: Are all required fields populated?
  - Structural compliance: Does output match schema? (rule-based)
  - Tone adherence: BLEU/ROUGE vs reference documents
  - Legal phrase preservation: Critical legal clauses present (keyword check)

Escalation Metrics:
  - Precision: true positives / (true + false positives)
  - Recall: true positives / (true + false negatives)
  - F1 score: target > 0.90
```

---

### 6.2 Online Metrics

**Real-time metrics collected per request:**

```
Performance:
  - e2e_latency_ms (p50, p95, p99 by workflow type)
  - ttfb_ms (time to first byte for streaming)
  - retrieval_latency_ms
  - model_call_latency_ms

Quality (sampled, 10% of traffic):
  - hallucination_flag: LLM-as-judge async eval on sampled responses
  - user_feedback: thumbs up/down captured in UI
  - escalation_rate: fraction of Q&A sessions that escalated
  - doc_revision_rate: fraction of generated docs sent back for revision

Cost:
  - tokens_input, tokens_output per request
  - cost_usd per request
  - cost_per_workflow_type (Q&A vs doc_gen vs escalation)
  - daily_cost_burn per tenant

Reliability:
  - retrieval_empty_rate: fraction of queries with 0 chunks returned
  - model_error_rate: 4xx/5xx from model providers
  - fallback_invocation_rate
```

---

### 6.3 Prompt & Model Versioning for A/B Rollout

```
Experiment framework (LaunchDarkly / internal flag system):

experiment_config:
  id: "sonnet4-vs-gpt4o-docgen"
  start_date: "2025-01-15"
  traffic_split:
    control: 50%  → model: claude-sonnet-4, prompt: v2.1.0
    treatment: 50% → model: gpt-4o-2024-11-20, prompt: v2.1.0
  tenant_eligibility: "enterprise"
  metrics_to_track:
    - doc_revision_rate (primary)
    - cost_per_request (guardrail: must not increase > 20%)
    - latency_p95 (guardrail: must not increase > 500ms)
  stopping_rules:
    - significant_harm: auto-stop if hallucination_rate > 10%
    - statistical_significance: min 500 samples per arm
```

**Versioning contract:**
- Every model call tags response with `model_id` + `prompt_version` + `experiment_id`
- Results queryable in data warehouse for offline analysis
- A/B decision reviewed by ML team + HR product owner before full promotion

---

## 7. Failure Modes & Fallbacks

### 7.1 Partial Retrieval Failure

**Scenario:** Vector DB returns < 2 chunks, or all chunks have score < 0.5 threshold.

```
Detection: retrieval_service checks chunk count and min_score before returning
Response strategy:
  1. Attempt BM25 sparse retrieval independently
  2. If still insufficient: respond with "I don't have enough information in our
     policy documents to answer this confidently. Here is what I found: [partial].
     Please consult your HR team for a definitive answer."
  3. Tag response as low_confidence=true → captured in metrics
  4. If retrieval service itself is down (timeout > 2s):
     → Fall back to zero-retrieval: model answers from parametric knowledge
     → Clearly label: "Note: This answer is from general knowledge, not your
       company's specific policies. Please verify with HR."
     → Escalation offered proactively
```

---

### 7.2 Model Hallucination

**Detection layers:**

```
Layer 1 — Grounding check (real-time, lightweight):
  - Extract factual claims from response using fast model
  - Cross-check against retrieved chunks (semantic similarity)
  - Flag if claim not found in any chunk with similarity > 0.7

Layer 2 — Confidence calibration:
  - If model outputs "I'm not sure" or hedging language → flag for review
  - If response contradicts a retrieved chunk directly → flag + suppress

Layer 3 — Async LLM-as-judge (sampled 10%):
  - Send (query, retrieved_chunks, response) to evaluation model
  - Returns: {"faithful": true/false, "unsupported_claims": [...]}
  - Feeds into quality dashboard
```

**Fallback on detected hallucination:**
1. Suppress response; ask model to regenerate with stricter grounding instruction
2. If second attempt also flagged: return "I'm unable to provide a confident answer based on available policies. Please escalate to HR."
3. Log event with full context for audit

---

### 7.3 External Tool / Provider Failure

**LiteLLM-level fallback chain:**

```yaml
model_fallback_chain:
  primary:
    model: claude-sonnet-4
    timeout_s: 10
    max_retries: 2
  fallback_1:
    model: gpt-4o-2024-11-20
    timeout_s: 10
  fallback_2:
    model: claude-haiku-4-5  # Faster, cheaper, degraded quality
    timeout_s: 5
    note: "Inform user: degraded mode, simpler answers only"
  fallback_3:
    mode: static_fallback
    action: "Return pre-written FAQ answers for top 50 questions from cache"
    action_escalation: "For other queries: queue for async HR response within 4hrs"
```

**Circuit breaker pattern:**
- If provider error rate > 10% in last 60s: open circuit, skip to next fallback immediately
- Circuit re-checks every 30s (half-open state)
- Alerts fired to on-call when circuit opens

---

### 7.4 Additional Failure Scenarios

| Failure | Detection | Response |
|---|---|---|
| PII redaction service crash | Healthcheck timeout | Block all LLM calls until restored; queue requests |
| Embedding service down | Retrieval timeout | Degrade to BM25-only retrieval; log degraded mode |
| Kafka unavailable | Write failure on audit event | Write to local DB buffer; replay on reconnect |
| Context too long (overflow) | Token count check pre-call | Truncate conversation history first; then retrieved chunks |
| Sensitive topic with no policy | Classification score | Auto-escalate with summary of what user asked |
| Tenant cost cap exceeded | Budget check in orchestrator | Return 429 with message; alert HR admin; queue doc gen |

---

## 8. Cost Control Architecture

### 8.1 Multi-Layer Cost Enforcement

```
Layer 1: Request-level (Orchestrator)
  - Token budget enforced before model call
  - Intent routing to cheapest capable model
  - Q&A can use Haiku if query is simple (confidence-scored)

Layer 2: Session-level (API Gateway)
  - Per-session token cap (prevents runaway conversations)
  - Streaming cutoff if output exceeds budget mid-stream

Layer 3: Tenant-daily (LiteLLM + DB)
  - Check tenant.daily_cost_spent before every model call
  - If > 90% of cap: alert, degrade to cheaper models
  - If 100% cap: block further model calls, serve cached/static responses

Layer 4: System-level (Monthly)
  - Cloud billing alerts at 50%, 80%, 100% of monthly budget
  - Anomaly detection on per-tenant spend spikes (z-score > 3)
```

### 8.2 Caching Strategy

```
Cache types:
  1. Semantic cache (Redis): hash(embedding(query)) → cached response
     - Hit rate target: 30% for Q&A (policy questions recur)
     - TTL: 24 hours (invalidated on policy document update)
     - Freshness check: if policy effective_date > cache timestamp, invalidate

  2. Tenant context cache (Redis): tenant_id → context blob
     - TTL: 1 hour
     - Invalidated on admin config change

  3. Embedding cache (Redis): text_hash → embedding vector
     - TTL: 7 days
     - Used during re-indexing to avoid redundant API calls
```

---

## 9. Data Flow Diagrams

### 9.1 Policy Q&A Flow (Happy Path)

```
Employee asks: "How many days of sick leave do I get per year?"

1. API Gateway: authenticate JWT, inject tenant_id=acme, user_role=employee
2. Orchestrator → Intent Classifier: intent=qa, confidence=0.97, sensitivity=low
3. PII Redactor: no PII detected (clean pass)
4. Context Builder:
   - System prompt v2.3.0
   - Tenant: Acme Corp, India jurisdiction, FY2025 policies active
   - User role: employee (no salary/perf context included)
5. Retrieval Service:
   - Dense: top-5 chunks from "acme/leave_policy_2025.pdf"
   - BM25: top-5 chunks
   - RRF fusion: top-3 chunks selected (scores: 0.91, 0.87, 0.72)
6. Context assembled: 3,200 tokens total
7. Model call → Claude Sonnet: "Based on Acme's Leave Policy (effective Jan 2025),
   you are entitled to 12 days of sick leave per calendar year..."
8. Grounding check: claim "12 days" found in chunk_id 3a7f → pass
9. Response streamed to UI with citations: [Leave Policy 2025, Section 4.2]
10. Audit log written: {trace_id, cost=$0.0032, latency=1840ms, chunks=3}
```

---

### 9.2 Document Generation Flow

```
HR Admin: "Generate offer letter for Senior Engineer, ₹28L CTC, Bangalore"

1. RBAC check: role=hr_admin → doc_gen permitted
2. Intent: doc_gen, doc_type=offer_letter
3. PII Redactor: salary amount flagged → tokenized for doc template only
4. Template Engine: selects offer_letter_india_v3.docx template
5. Variable extraction: {role: "Senior Engineer", location: "Bangalore",
   ctc: [TOKEN:salary_1], joining_date: null → prompt user}
6. Clarification: "What is the joining date?" → user responds "March 1, 2025"
7. Model call (Claude Sonnet): fill template prose sections, generate role
   responsibilities summary based on JD (retrieved from doc store)
8. Compliance check: mandatory clauses present (POSH, NDA, IP assignment) ✓
9. Human review gate: doc flagged for HR admin approval before download
10. Draft returned as Word doc preview in editor
11. Admin reviews, approves, downloads signed-off offer letter
12. Audit: generated_docs record created, approval event logged
```

---

## 10. Implementation Roadmap

### Phase 1 — Foundation (Weeks 1–6)

| Week | Deliverable |
|---|---|
| 1–2 | API Gateway + Auth; PostgreSQL schema + RLS; basic RBAC |
| 3–4 | Retrieval service: document ingestion pipeline, vector DB setup, BM25 index |
| 5–6 | Orchestrator core: intent classifier, context builder, basic Q&A workflow |

**Exit criteria:** Policy Q&A end-to-end working for single tenant, with audit logging and PII redaction.

---

### Phase 2 — Core Features (Weeks 7–12)

| Week | Deliverable |
|---|---|
| 7–8 | Document generation workflow + template engine |
| 9–10 | Escalation workflow + HRBP inbox |
| 11–12 | Golden dataset construction + offline eval pipeline |

**Exit criteria:** All three workflows functional; hallucination detection in place; offline eval passing.

---

### Phase 3 — Production Hardening (Weeks 13–18)

| Week | Deliverable |
|---|---|
| 13–14 | Cost control: budget enforcement, semantic caching, LiteLLM fallback chain |
| 15–16 | Full observability: Grafana dashboards, online metrics sampling, alerting |
| 17–18 | A/B framework; multi-tenant isolation validation; penetration test |

**Exit criteria:** SLOs defined and monitored; first enterprise tenant onboarded; security audit passed.

---

### Phase 4 — Scale & Optimize (Weeks 19–24)

| Week | Deliverable |
|---|---|
| 19–20 | Fine-tuned sensitivity classifier deployed (replaces prompt-based) |
| 21–22 | Self-hosted embedding model for data-residency tenants |
| 23–24 | Continuous eval pipeline (nightly golden dataset regression) |

---

## Appendix: Interview Evaluation Guide

### What a Strong Candidate Covers

**Architecture (Components):**
- Separates retrieval from generation clearly
- Mentions tenant isolation at every data layer (not just DB)
- Identifies LiteLLM or similar as the model abstraction layer
- Considers the async vs sync nature of different workflows

**Context Engineering:**
- Knows what NOT to include (PII, salary in most roles)
- Mentions context versioning as a deployment concern
- Understands token budget tradeoffs

**Model Selection:**
- Does not default to "just use GPT-4 for everything"
- Correctly identifies RAG > fine-tuning for time-varying policy knowledge
- Mentions fine-tune only for taxonomy/classification tasks

**Security:**
- Mentions PII at API boundary, not just in DB
- Raises ZDR agreements with external providers
- Discusses RBAC at multiple layers (gateway, orchestrator, DB)

**Observability:**
- Distinguishes offline (golden set) from online (sampled LLM-as-judge) evaluation
- Mentions hallucination detection as an active problem, not solved
- Discusses cost observability as a first-class concern

**Failure Modes:**
- Covers graceful degradation (not just hard failures)
- Discusses user-facing messaging for degraded states
- Mentions circuit breakers for external dependencies

### Red Flags

- No mention of PII handling until prompted
- Assumes single-tenant architecture
- Proposes fine-tuning for policy Q&A (stale model problem)
- No cost controls ("we'll add that later")
- Treats hallucination as solved by retrieval alone
- No versioning strategy for prompts or models

---

*Document version: 1.0.0 | Last updated: April 2026 | Prepared for: HR SaaS Principal Engineer System Design Review*
