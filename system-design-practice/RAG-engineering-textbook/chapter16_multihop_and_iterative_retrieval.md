# Chapter 16: Multi-hop & Iterative Retrieval

## 16.1 What This Chapter Covers

Every retrieval strategy in Chapter 14, and every re-ranking technique in Chapter 15, shares one assumption: that a single retrieval pass, over a single (possibly rewritten or expanded) query, can surface the information needed to answer the question. For a large share of real queries, that assumption holds fine. But some questions simply cannot be answered this way, no matter how good your embeddings, your index, or your re-ranker are — because the answer doesn't live in any one place.

This chapter is about that harder category of question, and the retrieval pattern built to handle it: retrieving, reasoning about what you found, and retrieving again — as many times as it takes.

---

## 16.2 The Question a Single Retrieval Pass Can't Answer

Consider: "Who is the CEO of the company that acquired the company Jane worked for in 2019?"

Think about what it would take to answer this correctly. There is almost certainly no single document in your knowledge base that states this fact directly — because it isn't really *one* fact. It's a chain:

1. Find where Jane worked in 2019.
2. Find what company acquired *that* company.
3. Find who the current CEO of the acquiring company is.

Each step depends on the answer to the previous one. A single query embedding for the whole question is a blurry average of all three sub-needs at once, and even a well-tuned retriever (Chapter 14) or a sharp query decomposition (Chapter 13) that splits it into three sub-questions up front runs into a problem: **you don't actually know what step 2's query should be until you've answered step 1.** You can't write "what company acquired Jane's 2019 employer" as a search query until you know the name of that employer.

This is the defining feature of what's called a **multi-hop question**: answering it requires a *sequence* of lookups, where later lookups depend on facts discovered in earlier ones, rather than a set of independent sub-questions that could all be issued in parallel.

> Multi-hop retrieval exists for questions where the retrieval query itself cannot be fully specified in advance — because part of that query is a fact you don't have yet, and can only get by retrieving first.

This is a step beyond the decomposition we introduced in Chapter 13. Decomposition there handled *known, independent* sub-questions issued up front. Multi-hop retrieval handles *dependent, discovered-as-you-go* sub-questions — and that dependency is exactly what forces retrieval to become an iterative process instead of a single pass.

---

## 16.3 The Iterative Retrieve-Reason-Retrieve Loop

The general pattern for handling multi-hop questions is a loop, rather than a pipeline. At a high level:

```
Question
   │
   ▼
[1] Retrieve  ──►  search based on current knowledge
   │
   ▼
[2] Reason    ──►  does this answer the full question, or reveal a
   │                new fact needed to continue?
   │
   ├── Enough to answer ──► [4] Generate final answer
   │
   └── Need more info
          │
          ▼
   [3] Formulate next query using newly discovered facts
          │
          └──────────────► back to [1] Retrieve
```

Walking through our example:

- **Hop 1:** Retrieve for "where did Jane work in 2019." Suppose this surfaces a document stating Jane worked at "Northwind Analytics" in 2019.
- **Reasoning step:** The system now knows the employer's name — a fact it didn't have when the question was first asked — and recognizes it still needs the acquirer.
- **Hop 2:** Retrieve for "company that acquired Northwind Analytics." Suppose this surfaces that Northwind Analytics was acquired by "Vantage Corp" in 2021.
- **Reasoning step:** Now it needs Vantage Corp's current CEO.
- **Hop 3:** Retrieve for "current CEO of Vantage Corp." This surfaces the final fact.
- **Generation:** With all three facts now retrieved, the system generates the final answer, ideally citing all three source documents.

Each hop's query is only knowable *after* the previous hop's retrieval completed and was reasoned over. This reasoning step — deciding whether enough information has been gathered, and if not, what to search for next — is typically performed by an LLM prompted to inspect the retrieved content and either produce a final answer or produce a follow-up query. This is the same mechanism, applied to retrieval specifically, that underlies the broader idea of **agentic RAG**, covered fully in Chapter 21: an LLM acting as a controller that decides what action to take next (in this case, "retrieve again, with this new query") rather than following a fixed, predetermined sequence of steps.

---

## 16.4 How the Loop Decides When to Stop

A natural question: what stops this loop from going forever? In practice, iterative retrieval systems use a combination of stopping conditions:

- **The reasoning step judges the question fully answered** — the LLM, inspecting everything retrieved so far, determines it has enough to respond and stops issuing new queries.
- **A maximum hop count** — a hard ceiling (e.g., 3–5 hops) prevents runaway loops on questions the system can't resolve, trading potential completeness for a predictable latency and cost bound.
- **No new information gained** — if a hop's retrieval returns content that's redundant with what's already been found, that's a signal the loop isn't making progress and should terminate rather than repeat.

Getting this stopping logic right is genuinely one of the harder parts of building these systems. Stop too early, and you return an incomplete answer with false confidence. Stop too late (or never), and you've built a system that's slow, expensive, and occasionally stuck.

---

## 16.5 Graph-Based Retrieval: An Alternative Path Through the Same Problem

Iterative retrieval treats each hop as an independent search over unstructured documents. There's a different way to attack the same class of question: if you pre-build a **knowledge graph** — entities (people, companies, products) connected by explicit, labeled relationships (works-at, acquired-by, CEO-of) — then a multi-hop question can potentially be answered by *traversing* the graph directly, following edges from Jane → Northwind Analytics → Vantage Corp → CEO, rather than issuing a sequence of open-ended document searches and hoping each one surfaces the right fact.

This is the foundation of **GraphRAG**, which we cover in full in Chapter 22. We mention it here because it's worth seeing multi-hop retrieval and graph-based retrieval as two answers to the same underlying problem, with a real tradeoff between them: iterative document retrieval requires no special data structure beyond your existing index, but each hop carries the uncertainty of open-ended search — the right document might not surface. Graph traversal is far more precise and reliable *if* the relevant facts and relationships were already extracted into the graph ahead of time, but building and maintaining that graph (an ingestion-time cost, tying back to Chapter 7's metadata extraction) is substantial additional engineering investment that not every knowledge base justifies.

---

## 16.6 Cost, Latency, and Complexity Tradeoffs

It's worth being direct about what multi-hop retrieval costs relative to the single-pass strategies in Chapter 14, because the difference is not small:

| Dimension | Single-pass retrieval | Multi-hop / iterative retrieval |
|---|---|---|
| **Retrieval calls per question** | 1 (or a fixed few, for multi-query) | Variable, often 2–5+, unknown in advance |
| **LLM reasoning calls** | 0–1 (query rewriting) | One per hop, to decide what's needed next |
| **Latency** | Low, predictable | Higher, and variable — proportional to hop count |
| **Cost** | Low, predictable | Higher, and variable, for the same reason |
| **Failure surface** | Bad retrieval, bad generation | All single-pass failure modes, compounded across hops, plus the loop terminating too early or too late |

That last row deserves emphasis: multi-hop retrieval doesn't just add cost, it **compounds risk**. If hop 1 retrieves the wrong employer for Jane, every subsequent hop is now chasing the wrong thread entirely, and the final answer will be confidently, coherently wrong — an error that's often harder to catch than a single bad retrieval, precisely because the reasoning connecting the hops looks sound even when the underlying fact was not.

This is why multi-hop retrieval should be treated as a targeted tool, not a default. Most production RAG systems benefit from detecting *which* questions genuinely require multiple hops — often via the same reasoning step used mid-loop, applied once up front — and routing only those questions into the more expensive iterative path, while everything else takes the cheaper single-pass route from Chapter 14.

---

## 16.7 A Note of Honesty: Iteration Is Not a Guarantee of Correctness

It's tempting to treat "just let the model retrieve again if it's not sure" as a general-purpose fix for weak retrieval. It isn't, for a few concrete reasons:

- **The reasoning step can be wrong about being done.** An LLM can judge (incorrectly) that it has enough information and stop early, producing a confidently incomplete answer with no visible sign anything was missed.
- **Errors compound across hops**, as noted above — there's no built-in mechanism to catch a wrong intermediate fact before it propagates forward.
- **Latency and cost scale with question difficulty in a way single-pass systems don't**, which makes multi-hop retrieval a poor fit for latency-sensitive, high-volume applications unless carefully scoped to only the queries that need it.
- **Evaluation gets harder.** Measuring whether a multi-hop answer is correct means tracing correctness through every hop, not just checking the final output — a challenge we'll pick up again in Part VII.

Multi-hop and iterative retrieval extend what RAG can answer, from single-document lookups to genuinely chained, multi-fact reasoning. They don't make retrieval infallible — they just move the same fundamental challenges (wrong documents, misread content, hallucination) into a longer, more consequential sequence.

---

## 16.8 Chapter Summary

- **Multi-hop questions** require chaining facts across multiple documents in sequence, where later retrieval queries depend on facts discovered in earlier ones — they cannot be fully specified, or answered, in a single retrieval pass.
- The core pattern is an **iterative retrieve-reason-retrieve loop**: retrieve, evaluate whether the question is answerable yet, and if not, formulate and issue a follow-up query based on newly discovered facts.
- This reasoning-and-control step is typically performed by an LLM acting as a controller — the same underlying idea that generalizes into **agentic RAG**, covered in Chapter 21.
- Loops need explicit stopping conditions (judged completeness, a maximum hop count, or lack of new information) to avoid running indefinitely.
- **Graph-based retrieval (GraphRAG, Chapter 22)** offers an alternative path to the same goal — traversing pre-built entity relationships instead of issuing sequential open-ended searches — trading ingestion-time engineering effort for more reliable multi-hop traversal.
- Multi-hop retrieval carries real costs beyond single-pass retrieval: higher and less predictable latency and cost, and compounding risk, since an error in an early hop propagates, confidently and often invisibly, into every hop that follows.
- Because of these costs, multi-hop retrieval works best as a targeted path for questions that genuinely need it, not a default applied to every query.

**Coming up next (Chapter 17):** with Part IV's retrieval toolkit complete — query understanding, retrieval strategies, re-ranking, and multi-hop retrieval — we turn to Part V and the generation side of RAG, starting with how to prompt an LLM effectively once it's holding a set of retrieved documents in hand.
