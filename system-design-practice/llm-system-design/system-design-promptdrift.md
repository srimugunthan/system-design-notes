# System Design Interview Problem: Prompt Tuning for Multi-Tenant Document Extraction

Pitched at Principal-level depth — ambiguous enough to require requirements-gathering, with room to go deep on the ML mechanics you'd be expected to know cold.

---

## Problem Statement (as given to candidate)

> Your company runs a document extraction service that pulls structured fields (amounts, dates, counterparty names, account numbers) from financial documents — invoices, KYC forms, loan agreements — for 50+ enterprise clients. Each client's documents have different layouts, terminology, and field conventions. You currently use a single frozen 13B instruction-tuned LLM with a hand-written prompt per client, maintained by an ML team of 3. This doesn't scale — every new client takes 2 weeks of prompt engineering, prompts drift out of sync with model updates, and accuracy varies wildly across clients (68%–94% field-level F1).
>
> Design a system that uses **prompt tuning** (not full fine-tuning) to give each client a customized extraction behavior, at a fraction of the engineering cost, while sharing one base model across all clients.
>
> You have 45 minutes.

---

## What a strong candidate should surface (interviewer's rubric)

### 1. Clarifying questions (first 5–10 min)
A candidate should probe before designing:
- Is this extractive (spans from source) or generative (model writes the answer)? Changes the loss function and eval story.
- Latency/throughput SLA — batch nightly extraction vs. real-time API?
- Is the base model served via an API you control (can you touch embeddings/logits) or a third-party API (GPT-4-class) where soft prompts are impossible?
- Do you have labeled data per client, and how much — 50 examples? 5,000?
- Compliance constraints — can client documents leave a VPC for training? (Common wall in financial services — good candidates raise it unprompted.)
- New-client cold-start requirement — how fast must a brand-new client get *some* working extraction?

This last one is the crux: it forces the candidate to design for **both** soft-prompt training *and* a fallback/bootstrap path.

### 2. Core architecture

Expect something like:

```
                    ┌─────────────────────────┐
                    │   Frozen Base LLM        │
                    │   (shared, one copy in   │
                    │    memory across tenants)│
                    └───────────▲──────────────┘
                                │
        ┌───────────────────────┴───────────────────────┐
        │            Per-client soft prompt store         │
        │  client_001: [v1,v2,...,vk] (learned embeddings)│
        │  client_002: [v1,v2,...,vk]                      │
        │  ...                                             │
        └───────────────────────┬───────────────────────┘
                                │
                    ┌───────────▼──────────────┐
                    │  Prompt Tuning Trainer    │
                    │  (per-client, offline)    │
                    │  - freezes base model     │
                    │  - backprop only into     │
                    │    virtual token embeds   │
                    └───────────▲──────────────┘
                                │
                    ┌───────────┴──────────────┐
                    │  Labeled extraction pairs │
                    │  per client (doc → fields)│
                    └───────────────────────────┘
```

Key design points to push on:

- **Batched inference across tenants**: since the base model weights are shared and only the prepended soft-prompt embeddings differ, you can batch requests from *different clients* in the same forward pass if your serving stack supports per-sequence prefix injection — this is the actual cost win over per-client fine-tuned copies. Ask the candidate to reason about GPU memory: one 13B model in memory + N tiny (k × d_model) embedding tables is vastly cheaper than N fine-tuned 13B copies.
- **Prompt length (k) as a hyperparameter**: tradeoff between expressiveness and data efficiency — longer soft prompts need more labeled examples to train well (Lester et al. showed prompt tuning needs scale to match fine-tuning; at 13B this is borderline, worth discussing).
- **Cold start for new clients**: candidate should propose either (a) prompt transfer/initialization from the nearest existing client's soft prompt (cluster by document type similarity) or (b) falling back to a hand-written discrete prompt + few-shot until enough labeled data accumulates to train a soft prompt, then cut over.
- **Training data loop**: where do labels come from — human-in-the-loop correction UI feeding back into a retraining queue? Active learning to prioritize which documents get human review?

### 3. Failure modes and mitigations — this is where seniority shows

- **Silent degradation on model upgrade**: if the frozen base model is swapped (v2 → v3), every client's soft prompt is now tuned against a distribution the model was never updated on — soft prompts don't transfer across base model versions the way discrete prompts loosely do. Candidate should propose a retraining pipeline gated on base-model version, and a shadow-eval step before cutover.
- **Per-client eval and regression testing**: 50 clients means 50 held-out sets; a bad soft-prompt retrain for client A shouldn't silently ship. Propose per-tenant eval gates in CI (ties back to Module 6 evaluation loop).
- **Data leakage / compliance**: soft prompt embeddings are derived from client documents — do they count as "client data" for compliance/residency purposes? Strong candidates flag this explicitly, given financial services context, even without being prompted.
- **Multi-tenant security**: could client A's soft prompt somehow leak information about client B in a shared-batching serving setup? Worth a sentence even if the answer is "no, they're just concatenated embeddings, no cross-tenant attention" — shows the candidate understands the mechanism, not just the buzzword.

### 4. Deeper follow-up questions the interviewer can escalate to

1. "Client accuracy is stuck at 75% even after tuning a 20-token soft prompt with 2,000 examples. What do you try next?" — Expects: check if the task is representable in-context at all (maybe base model lacks the domain knowledge, no amount of soft prompt tuning fixes a genuine capability gap); consider P-Tuning v2 (prefix at every layer, not just input) for more capacity; or graduate that client to LoRA/full fine-tuning if soft prompts have hit their ceiling — good candidates know prompt tuning has a **known capacity ceiling** relative to fine-tuning at smaller model scales.
2. "How would you A/B test a new soft prompt against production without risking client-facing accuracy?" — Expects shadow traffic / canary per tenant, not global rollout.
3. "A client wants to know why a field was extracted wrong. How do you debug a soft prompt, versus a discrete prompt?" — This is the real gotcha: soft prompts are **not interpretable** — you can't read the learned embeddings as text. Expects candidate to propose auxiliary tooling (nearest-neighbor decoding of virtual tokens back to vocabulary space, attention visualization) or to argue for keeping a discrete-prompt fallback specifically for the sake of debuggability/auditability in a regulated industry — this is a legitimate reason to *not* use pure prompt tuning end-to-end.
4. "Estimate the cost delta between this system and per-client full fine-tuning at 50 clients, 13B params." — Tests whether candidate can do rough back-of-envelope math on adapter storage vs. full model storage/serving costs.

---

## Why this problem works well as an interview question

- It has a legitimate "ambiguous requirements" phase, not just an architecture dump.
- The interpretability gotcha (#3 above) is a genuine, non-obvious limitation of soft prompts that separates candidates who've actually implemented prompt tuning from those who've only read the abstract.
- It naturally connects to your own Shield-Fin / AuditAgent portfolio work — you could reuse this almost verbatim as a talking point in your own interviews, framed as "here's a design problem I've thought through," since it touches guardrails, auditability, and multi-tenant financial services constraints directly.

Want me to write up a model answer / reference solution doc for this (as if you were the candidate), or turn this into a second problem focused on the **discrete/gradient-free optimization** side (OPRO/DSPy) instead of soft prompts, to cover both halves of the syllabus?

--
Here are 5 solid questions to explore around prompt engineering:

1. **How do you structure a prompt to reliably get structured output (JSON, XML) from an LLM, and what do you do when the model still deviates from the schema?**

2. **What's the difference between few-shot prompting and fine-tuning for a given task, and how do you decide which one to use given constraints like latency, cost, and data availability?**

3. **How do you debug a prompt that works well on some inputs but fails on edge cases — what's your systematic approach to isolating the failure mode?**

4. **How does chain-of-thought or step-by-step reasoning in a prompt affect output quality, and are there cases where it actually hurts performance or increases cost without benefit?**

5. **How do you evaluate and version prompts in production — what metrics or test sets do you use to catch regressions when you tweak a prompt?**

If you want, I can also tailor a set specifically around prompt engineering for financial services use cases (e.g., guardrails, red-teaming prompts, structured extraction from financial documents) given the work you've been doing with Shield-Fin and FinVision — just let me know.
