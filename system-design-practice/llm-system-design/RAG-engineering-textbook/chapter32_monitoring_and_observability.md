# Chapter 32: Monitoring & Observability

## 32.1 What This Chapter Covers

A normal web service is more or less easy to monitor: track latency, error rate, and throughput, and you have a decent picture of health. A RAG system can look perfectly healthy on every one of those dashboards — fast responses, zero errors, high uptime — while quietly giving wrong answers to a growing share of users. Nothing in a standard ops dashboard would tell you that.

This chapter asks: **what does it actually mean to observe a RAG system, given that its failures are usually failures of correctness, not failures of infrastructure?** We'll look at retrieval tracing, drift detection, and feedback loops — the three pieces that turn "the system is up" into "the system is still giving good answers."

---

## 32.2 Why RAG Needs More Than Standard Observability

Standard service monitoring answers questions like: *Is it up? Is it fast? Is it erroring?* These questions matter for RAG too, and Chapter 30's latency budgets and standard error-rate tracking still apply directly.

But RAG has a whole category of failure that produces a **200 OK response with a wrong answer**. The retriever fetched irrelevant chunks, but the model still confidently wrote something fluent. The knowledge base has a stale document that contradicts a newer one, and the model picked the wrong one. None of this trips an error handler. The request "succeeds" by every infrastructure metric while failing at the one thing the system exists to do.

> **Core idea:** in a RAG system, "no errors" and "correct answers" are almost entirely independent signals — you need to monitor both, and standard infrastructure tooling only gives you the first one.

This means RAG observability needs to answer a second layer of questions: *What did the retriever actually fetch, and was it relevant? What did the model actually see, and does the answer reflect it? Is quality slipping over time, even without a single alert firing?*

---

## 32.3 Retrieval Traces — Logging What Actually Happened

The single most valuable observability artifact in a RAG system is the **retrieval trace**: a complete, linkable record of everything that happened for one request, from the raw query to the final answer.

A good retrieval trace includes:

| Field | Why it matters |
|---|---|
| Request ID | Ties every downstream artifact together for debugging |
| Raw user query | What was actually asked |
| Rewritten/expanded query (Ch. 13) | What was actually searched, if different from the raw query |
| Retrieved chunks (IDs, source doc, scores) | What the retriever found and how confident it was |
| Re-ranked order and scores (Ch. 15) | Whether re-ranking changed the picture, and by how much |
| Final assembled prompt | Exactly what the model saw — the ground truth for debugging generation issues |
| Generated answer | What the model produced |
| Latency per stage | Ties back to Chapter 30's budget for performance debugging |
| Model/index/prompt versions | Which version of everything produced this result |

The reason to log all of this — not just the final answer — is that debugging a bad answer without a trace is close to guesswork. If a user reports "this answer was wrong," the trace lets you immediately distinguish between very different root causes: Did the retriever fail to find the right document? Did it find the right document but rank it too low to make the cut? Did the right chunk make it into the prompt, but the model ignored or misread it? Each of those points to a completely different fix — better chunking, a re-ranking threshold change, or a prompt engineering fix — and without the trace, you're left re-running the query and hoping you can reproduce the failure.

Traces are also what make the evaluation and annotation work from Chapter 29 possible at scale: a human reviewer annotating "was this answer good?" is far more useful when they can also see *why* the system produced it, not just the final text.

A practical note: retrieved chunk content and prompts often contain the same sensitive information as the underlying documents. Retrieval traces need the same access controls as the documents they reference — a debugging tool that leaks restricted data to whoever has trace access is not a debugging win. This connects directly to Chapter 34's access-control discussion.

---

## 32.4 Drift Detection

Even a RAG system that was carefully evaluated and shipped in good shape can degrade silently over time, because two things underneath it keep moving: the knowledge base and the query distribution.

**Knowledge base drift.** Documents get added, updated, and — critically — sometimes should be removed but aren't (Chapter 33). Over months, the corpus composition shifts: a product line that used to dominate the knowledge base gets a smaller share of documents relative to a newer one, or a large batch of low-quality scraped content gets ingested without review. Retrieval quality that was tuned against last quarter's corpus may not hold against this quarter's.

**Query distribution drift.** What users ask changes as the product changes, as new features ship, or as the user base grows into a new segment. A retriever and prompt template tuned against last year's typical questions may perform noticeably worse against this year's, even with an unchanged knowledge base, simply because the query patterns have shifted away from what it was implicitly optimized for.

**Embedding and score drift.** If the embedding model, chunking strategy, or re-ranker is updated, the distribution of retrieval scores can shift even for the same underlying documents and queries — a similarity threshold that made sense with the old embedding model may be miscalibrated with the new one.

Practical signals worth tracking over time, not just at a point in time:

- Retrieval score distributions (median/percentile similarity scores per week or month)
- The rate of "no relevant results found" or empty-retrieval responses
- The proportion of queries hitting newly added vs. long-standing documents
- User feedback signal trends (Section 32.5) segmented by query type or topic
- Sampled trace review — a rotating manual audit of a small sample of traces, because some drift is genuinely easier for a human to notice than to encode as a metric

None of these need to trigger a hard alert threshold the way an error rate spike does. The goal is a trend view that a team looks at periodically, because RAG quality degradation is much more often a slow slope than a cliff.

---

## 32.5 Feedback Loops

The final piece is closing the loop: turning production usage back into signal that improves the system, rather than letting every trace disappear into a log that nobody revisits.

**Explicit feedback.** Thumbs up/down, star ratings, or "was this helpful?" prompts are the most direct signal, but they suffer from low response rates and self-selection bias — users who bother to click are disproportionately the very satisfied or very frustrated ones. Treat explicit feedback as a useful sample, not a representative one.

**Implicit feedback.** Often more abundant and less biased, though noisier to interpret:

- Did the user immediately rephrase and re-ask the same question? (Likely a bad first answer.)
- Did the user copy the answer, or take an action suggested by it?
- Did a follow-up conversation contradict or correct the previous answer?
- Did the user abandon the session right after the answer?

**Routing feedback into evaluation.** Feedback signals are most valuable when they feed directly into the evaluation and human annotation pipeline from Chapter 29 — a thumbs-down response, paired with its full retrieval trace, is an ideal candidate for a human reviewer to triage: is this a retrieval problem, a generation problem, or a genuinely hard/ambiguous question that no system would answer well? Systematically routing negative-signal traces to review, rather than relying only on periodic manual spot checks, is what turns monitoring from a passive dashboard into an active improvement loop.

```
Production traffic
      │
      ▼
Retrieval trace logged (Section 32.3)
      │
      ├──► Drift metrics (Section 32.4) ──► trend dashboards
      │
      └──► User feedback (explicit + implicit)
                  │
                  ▼
           Negative-signal traces routed to human review (Ch. 29)
                  │
                  ▼
           Feeds back into eval sets, re-ranking tuning, chunking fixes
```

---

## 32.6 A Note of Honesty — Observability Doesn't Fix Anything By Itself

It's worth being direct about a trap teams fall into: building excellent retrieval traces, drift dashboards, and feedback pipelines, and then not acting on any of it. Observability is diagnostic, not curative.

- **Traces tell you what happened, not what to do about it.** Turning "the re-ranker demoted the correct chunk" into a fix still requires the re-ranking work from Chapter 15.
- **Drift detection tells you quality is slipping, but rarely tells you exactly why**, especially when multiple things (corpus, queries, model version) are shifting at once. Isolating the cause usually still requires manual investigation.
- **Feedback signals are noisy and biased**, and treating a single week of thumbs-down spikes as ground truth without checking the underlying traces can send a team chasing the wrong fix.
- **All of this has a real cost** — storing full traces at scale, especially with sensitive content, is not free, and needs the same retention and access discipline as the source documents (Chapter 34).

The payoff of good observability is entirely dependent on a team actually reviewing the traces, watching the trends, and closing the loop back into evaluation. A monitoring system nobody looks at is just additional storage cost.

---

## 32.7 Chapter Summary

- Standard service monitoring (latency, errors, uptime) is necessary but not sufficient for RAG, because RAG's most important failures — wrong or unsupported answers — don't show up as errors.
- A **retrieval trace** should capture the full lifecycle of a request: raw and rewritten query, retrieved chunks with scores, re-ranked order, the final assembled prompt, the generated answer, and version metadata — all linkable by request ID.
- Traces are what make debugging a specific bad answer, and human evaluation at scale (Chapter 29), actually tractable — without them, root-causing a bad answer is guesswork.
- **Drift** happens along multiple axes — the knowledge base, the query distribution, and even score calibration after a model update — and tends to degrade quality slowly rather than suddenly.
- Useful drift signals include retrieval score trends, empty-result rates, and periodic sampled trace review, tracked over time rather than as one-time snapshots.
- **Feedback loops** — explicit (thumbs up/down) and implicit (rephrasing, abandonment, follow-up corrections) — are most valuable when routed directly into the evaluation and annotation pipeline, not left as a passive log.
- Observability is diagnostic: it tells you something is wrong and often where, but fixing it still requires the retrieval, chunking, and generation work covered elsewhere in this book — and it carries its own cost and access-control obligations.

**Coming up next (Chapter 33):** one of the biggest sources of the drift this chapter warns about is a knowledge base that's out of sync with its source systems — next we look at incremental indexing and how to keep a RAG system's data fresh without re-indexing everything from scratch.
