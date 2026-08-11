# Chapter 38: Case Studies

## 38.1 What This Chapter Covers

We have spent thirty-seven chapters taking RAG apart piece by piece: chunking in Chapter 6, hybrid search in Chapter 12, re-ranking in Chapter 15, evaluation in Chapters 26–29, security in Chapter 34. Now we put the pieces back together.

This chapter walks through three end-to-end RAG builds, each in a different domain, each starting from a naive first attempt and evolving into something that actually holds up in production. The goal isn't to hand you a blueprint to copy verbatim — your documents, your users, and your failure modes will be different. The goal is to show you *how the decisions from earlier chapters actually get made together*, under real constraints, and how a first attempt at RAG almost always needs at least one honest round of "this didn't work, here's why."

---

## 38.2 A Note on What These Case Studies Are

Before we go further, an important disclaimer: **the three case studies in this chapter are composite and illustrative.** They are not transcripts of real, named companies' engineering decisions. They are synthesized from patterns that show up repeatedly across enterprise search, customer support, and financial-services RAG deployments — patterns you will recognize if you've built any of these systems, and patterns that follow directly from the mechanics covered earlier in this book.

Think of them the way a case study in a business-school course works: the specifics are constructed to teach the mechanism clearly, not to document a particular company's history. Any resemblance to a specific product or organization is coincidental. Any numbers given (latency figures, accuracy percentages, team sizes) are illustrative orders of magnitude meant to convey scale and tradeoffs — not verified benchmarks.

With that framing in place, let's build three systems.

---

## 38.3 Case Study 1: Enterprise Search Over Wikis and Tickets

**The setup.** A mid-size software company (roughly 800 employees) wants a single search assistant that lets any employee ask a natural-language question and get an answer grounded in the company's internal wiki (product docs, onboarding guides, architecture decisions) and its support-ticket history (thousands of resolved tickets containing tribal knowledge that never made it into the wiki).

**First attempt.** The team stood up a simple pipeline in a few weeks: pull every wiki page and every closed ticket, split each document into fixed 512-token chunks, embed them with an off-the-shelf embedding model (Chapter 9), store them in a vector database (Chapter 10), and do pure vector similarity search (Chapter 14) on every query.

**What went wrong.** Two problems surfaced almost immediately in internal dogfooding:

1. **Fixed-size chunking cut through structure.** Wiki pages had tables, numbered runbooks, and code blocks. A 512-token window regularly split a step-by-step incident-response runbook into two chunks, so the retriever would return "step 4 of 7" with no context about what steps 1–3 were. This is the exact failure mode Chapter 6 warns about when it argues for structure-aware chunking over naive fixed windows.
2. **Pure vector search missed exact-match queries.** Employees frequently searched for error codes, internal service names, and ticket IDs — short, high-precision strings where semantic similarity is the wrong tool. "ERR_4402_TIMEOUT" doesn't need a model to understand its meaning; it needs an exact keyword hit. Vector-only retrieval buried these behind semantically-similar-but-wrong results.
3. **No access control.** Any employee's query could surface chunks from wiki spaces that were meant to be restricted (e.g., an HR-only compensation-planning page), because the ingestion pipeline had flattened everything into one shared index without carrying permission metadata forward.

**The fix.** The second iteration made three targeted changes:

- **Structure-aware chunking** (Chapter 6): chunk boundaries were made to respect headings, list boundaries, and code fences, with a maximum size cap rather than a fixed size, plus a small sliding overlap so runbook steps stayed together.
- **Hybrid search** (Chapter 12): the team added a keyword/BM25 index alongside the vector index and combined the two with reciprocal rank fusion. Exact-match queries (error codes, ticket IDs, service names) now surfaced correctly, while conceptual questions ("how do we handle SSO provisioning for new customers?") still benefited from semantic retrieval.
- **Access control carried through retrieval** (Chapter 34): document-level permissions from the source wiki and ticketing system were extracted as metadata at ingestion time (Chapter 7) and enforced as a *pre-filter* at query time, so a user's search never even considered chunks they weren't authorized to see — rather than retrieving them and hoping a prompt instruction would suppress them.

**Outcome.** After this second pass, the team also added a lightweight re-ranker (Chapter 15) in front of generation, since hybrid retrieval's top-20 candidates still needed sorting by actual relevance to the specific question asked. The lesson the team took away: the naive version of RAG is genuinely fast to build, and that speed is exactly what makes it tempting to ship before the structural and security gaps are found in production instead of in review.

---

## 38.4 Case Study 2: Customer Support RAG Bot for a SaaS Product

**The setup.** A SaaS company wants to deflect a portion of its support ticket volume by letting customers ask questions directly against product documentation, release notes, and a curated set of past support answers, with a clear path to a human agent when the bot can't help.

**First attempt.** The initial version followed the pattern from Chapter 3: retrieve relevant doc chunks, stuff them into a prompt, generate an answer. The prompt simply said "use the following context to answer the user's question."

**What went wrong.** Two failure modes emerged during a limited beta:

1. **Ungrounded confidence.** When retrieval returned weak or irrelevant chunks — which happened often for edge-case questions not well covered in the docs — the model still produced a fluent, confident-sounding answer by falling back on its parametric knowledge (Chapter 1). Several of these answers described features that didn't exist in the product, or described competitor products' behavior instead. Customers had no way to tell a grounded answer from a hallucinated one.
2. **No escalation path.** The bot always attempted an answer. There was no mechanism for it to recognize "I don't actually have good information for this" and hand off to a human, so customers with genuinely unsupported questions got a wrong answer instead of a fast escalation — arguably worse than no bot at all.

**The fix.** The team applied two changes drawn directly from later parts of the book:

- **Stricter grounding in the prompt** (Chapter 17): the prompt was rewritten to explicitly instruct the model to answer *only* from the provided context, to quote or cite the specific document section it drew from, and to say plainly when the context didn't contain an answer, instead of guessing. This alone cut down confidently-wrong answers substantially, though — consistent with the honesty theme running through this book since Chapter 1 — it did not eliminate them.
- **A corrective-retrieval and escalation layer** (Chapter 23): before generation, retrieved chunks were scored for relevance; if the top results scored below a threshold, or if the generated answer's self-assessed confidence was low, the system triggered a fallback — either a re-query with a reformulated question or a direct handoff to a human agent with the conversation context attached, rather than forcing the model to answer regardless.

The team then built out a proper evaluation loop (Chapters 26–29): a held-out set of real customer questions with known-good answers, retrieval metrics (precision/recall of the right doc being retrieved), generation metrics (faithfulness to the retrieved context, not just fluency), and a human-in-the-loop review process for a sample of live conversations each week. This evaluation harness is what let the team distinguish "the bot got better" from "the bot just sounds more confident," which — as Chapter 28 discusses — are not the same thing.

**Outcome.** Ticket deflection improved only after the escalation path was added — not before. The first version's raw "answer everything" approach actually generated *more* support burden in some cases, because agents then had to clean up after a wrong bot answer. The lesson: a RAG system that knows when to say "I'm not sure, let me connect you with someone" is often more valuable than one that always sounds sure of itself.

---

## 38.5 Case Study 3: Financial Compliance Q&A Assistant

**The setup.** A financial services firm wants an internal assistant that lets compliance analysts ask questions against regulatory filings, internal policy documents, and past compliance rulings — a domain where Chapter 35 already told us the stakes of a wrong answer are high, and where every answer may need to be defended to an auditor later.

**First attempt.** The team built a standard RAG pipeline over a document store that was refreshed with a full re-index whenever policy documents changed. Answers were generated with citations back to source documents, which felt sufficient at first.

**What went wrong.** The gap didn't show up in day-to-day use — it showed up during an internal audit:

1. **No way to reconstruct what the system knew at the time of a past answer.** Policies change quarterly. When an analyst's decision from four months ago was questioned, the team could not reliably reproduce *which version* of the policy document the assistant had actually retrieved and cited at that time, because the index had since been overwritten by newer versions with the same document ID. This is precisely the data-freshness and versioning gap discussed in Chapter 33.
2. **No durable audit trail.** The system logged final answers but not the full retrieval context, the model version, or the prompt used to produce them. For a regulated environment, "the answer was probably grounded in the right policy" is not an acceptable audit answer.

**The fix.** The redesign centered on treating the knowledge base as a versioned, append-only system rather than a mutable one:

- **Document versioning** (Chapters 33 and 35): each ingested policy document was stored with an effective-date range, and old versions were retained rather than overwritten. Incremental indexing added new versions without deleting the retrievability of prior ones.
- **Point-in-time retrieval:** the query interface accepted an "as of" date, so analysts (and later, auditors) could ask "what would the assistant have retrieved and answered on this date," reproducing historical answers rather than approximating them.
- **Full audit logging:** every query stored the exact retrieved chunks (with document version IDs), the assembled prompt, the model and prompt-template version, and the final answer — satisfying the traceability expectations Chapter 34 describes for regulated deployments.

**Outcome.** This version was noticeably more engineering-heavy than the first two case studies, and slower to ship — which is itself the point. In a regulated domain, the "boring" infrastructure work (versioning, logging, reproducibility) is not optional polish; it's the difference between a tool compliance will actually approve for use and one that creates new liability.

---

## 38.6 Cross-Cutting Lessons

Looking at all three case studies together, a pattern emerges that is worth naming explicitly:

| Case Study | First-attempt gap | What closed it | Chapters involved |
|---|---|---|---|
| Enterprise search | Naive chunking + vector-only search + no permission enforcement | Structure-aware chunking, hybrid search, permission-filtered retrieval | 6, 12, 34 |
| Customer support bot | Ungrounded answers, no escalation | Strict grounding prompts, corrective retrieval, evaluation loop | 17, 23, 26–29 |
| Compliance assistant | No versioning, no audit trail | Point-in-time document versioning, full query/answer logging | 33, 34, 35 |

None of these gaps were exotic. Each one is a direct, predictable consequence of skipping a technique this book covered in detail earlier — and each was invisible in a demo, surfacing only once real users, real edge cases, or real auditors showed up. That is the pattern worth internalizing more than any specific fix: **a working demo and a trustworthy system are different achievements**, and the distance between them is usually filled with exactly the unglamorous work — chunking discipline, retrieval hybridization, access control, evaluation harnesses, versioning, logging — that this book has spent thirty-seven chapters on.

---

## 38.7 What These Case Studies Don't Show

In the interest of the same honesty this book has tried to maintain since Chapter 1: these composite case studies are simplified. Real deployments involve organizational friction these narratives skip over — getting document owners to agree on what "the source of truth" even is, budget constraints that force cheaper embedding models or smaller context windows than you'd like (Chapter 31), and the fact that "fixed it in iteration two" often actually took several more painful iterations than a tidy chapter narrative suggests. They also don't show the cases where a fix in one dimension quietly made another worse — for instance, a stricter grounding prompt that reduces hallucination but also makes the bot needlessly refuse questions it could have answered. Real RAG engineering involves this kind of tradeoff constantly, and no single case study can fully capture it. Treat these three as illustrations of a method, not as a checklist guaranteed to work unmodified on your own system.

---

## 38.8 Chapter Summary

- This chapter presented three **composite, illustrative** case studies — not real named deployments — synthesizing common patterns across enterprise search, customer support, and financial compliance RAG systems.
- The **enterprise search** case showed how naive fixed-size chunking and vector-only retrieval break on structured documents and exact-match queries, fixed by structure-aware chunking, hybrid search, and metadata-enforced access control.
- The **customer support** case showed how ungrounded generation and a missing escalation path produce confidently wrong answers, fixed by strict grounding prompts, corrective retrieval, and a real evaluation loop.
- The **financial compliance** case showed how skipping document versioning and audit logging creates liability that only surfaces during an audit, fixed by point-in-time retrieval and full traceability logging.
- Across all three, the pattern is the same: **first attempts fail in predictable, previously-documented ways**, and the fixes map directly onto techniques covered earlier in this book.
- The gap between a working demo and a trustworthy production system is usually made of unglamorous engineering — chunking discipline, hybridization, evaluation, versioning, logging — not a single clever technique.
- Real deployments involve tradeoffs and organizational friction that no case study fully captures; treat these narratives as illustrations of method, not as guaranteed checklists.

**Coming up next (Chapter 39):** we've looked backward at how RAG systems get built and hardened in practice. In the final chapter of this book, we look forward — at long-context models, retrieval-augmented fine-tuning, real-time RAG over streaming data, and what's genuinely worth watching in a field that keeps moving quickly.
