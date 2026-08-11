# Chapter 23: Corrective RAG (CRAG) & Self-RAG

## 23.1 What This Chapter Covers

Every RAG system we've discussed so far shares a quiet assumption: whatever the retriever hands back gets passed to the generator, and the generator does its best with it. But what if the retriever hands back three chunks and only one is actually relevant? What if it hands back nothing useful at all? A standard pipeline doesn't notice — it generates an answer anyway, grounded in whatever mediocre context it received.

This chapter answers the question: **can a RAG system be taught to evaluate its own retrieval quality — and its own output — before committing to an answer, instead of blindly trusting whatever came back?**

---

## 23.2 Not All Retrieved Context Is Good Context

It's tempting to think of the retriever as binary: it either finds the answer or it doesn't. In practice, retrieval quality is a spectrum, and the failure modes are subtle:

- **Fully relevant** — the retrieved chunks directly answer the question
- **Partially relevant** — some retrieved chunks are useful, others are noise or tangential
- **Superficially relevant** — chunks share keywords or topic with the query but don't actually answer it (a classic embedding-similarity trap)
- **Irrelevant** — the retriever simply missed, often silently, with no error thrown

A standard RAG pipeline treats all four of these the same way: stuff whatever came back into the prompt and generate. We already know from Chapter 19 that noisy or contradictory context degrades generation quality. The natural next question is: why not check the quality of what was retrieved *before* generating, rather than hoping the generator handles bad context gracefully?

That's the shared motivation behind two related but distinct techniques: **Corrective RAG (CRAG)**, which grades and corrects retrieval, and **Self-RAG**, which has the model reflect on both retrieval and its own generation as part of a single learned behavior.

---

## 23.3 Corrective RAG: Grade First, Then Decide

CRAG inserts an explicit grading step between retrieval and generation. Instead of retrieve-then-generate, the flow becomes retrieve-grade-correct-generate:

1. **Retrieve** as usual — run the query against the vector store (and any hybrid search, per Chapter 12)
2. **Grade** each retrieved chunk for relevance to the query — typically using a lightweight classifier or the LLM itself, scoring each chunk as roughly *correct*, *ambiguous*, or *incorrect*
3. **Take corrective action** based on the grades:
   - If retrieval is **confidently good** — proceed to generation using the retrieved chunks, often after a refinement step that strips out irrelevant sentences within otherwise-good chunks
   - If retrieval is **ambiguous** — combine what was retrieved with a supplementary action, such as a fallback web search, to fill the gap
   - If retrieval is **confidently bad** — discard the retrieved chunks entirely and fall back to an alternative source (web search is the canonical example in the original CRAG formulation) rather than generating from context known to be poor
4. **Generate** using the corrected, filtered context

> **Core idea:** don't just retrieve and hope — grade what you retrieved, and change your plan when the grade is bad, instead of feeding low-quality context to the generator and trusting it to compensate.

The grading step is the whole mechanism here, and it's worth being precise about what it is: a separate, focused judgment call — "is this chunk relevant to this query, yes or no (or maybe)" — that's much easier to get right than the full "write a correct answer" task. This is the same principle behind re-ranking in Chapter 15: a narrower, well-defined judgment tends to be more reliable than asking a model to do everything at once.

---

## 23.4 Self-RAG: Reflection Baked Into Generation

Self-RAG takes a related but architecturally different approach. Rather than bolting a grading step onto an existing pipeline, Self-RAG trains (or prompts) the model to emit explicit **reflection tokens** — structured judgments interleaved with its own generation — at several points in the process:

| Reflection point | Question the model asks itself | Example judgment |
|---|---|---|
| **Retrieve?** | Do I even need to retrieve for this? | `[Retrieve]` / `[No Retrieve]` |
| **Relevant?** | Is this retrieved passage actually relevant to the question? | `[Relevant]` / `[Irrelevant]` |
| **Supported?** | Is my generated answer actually backed by the retrieved passage, or am I going beyond it? | `[Fully Supported]` / `[Partially Supported]` / `[No Support]` |
| **Useful?** | Is this a genuinely useful response to the question? | scored on a scale |

Notice the first judgment: **whether to retrieve at all.** Not every query needs retrieval — "what's 2+2" or "rephrase this sentence more formally" doesn't benefit from a trip to the knowledge base, and retrieving anyway just adds latency and risks pulling in irrelevant context that confuses the generator. Self-RAG treats "should I retrieve?" as a first-class decision rather than an assumed step, which is the same instinct behind agentic RAG's tool-use loop in Chapter 21 — the difference is that here the decision is trained into the model's generation behavior rather than orchestrated by an external loop.

The second and third judgments are where Self-RAG earns its name: after generating a candidate answer, the model checks its own output against the source passage and labels whether the answer is actually **supported** by what was retrieved, or whether it has drifted into unsupported claims — a direct, built-in check against exactly the hallucination-despite-grounding failure mode we flagged as a limitation back in Chapter 1.

---

## 23.5 CRAG and Self-RAG Side by Side

These two approaches aren't competitors so much as two different points on the same spectrum of "teach the system to check itself."

| Aspect | CRAG | Self-RAG |
|---|---|---|
| **Where the check happens** | External step, between retrieval and generation | Interleaved within generation itself |
| **What's judged** | Retrieval quality only | Retrieval necessity, relevance, and output support |
| **Corrective action** | Explicit fallback (e.g., web search) when retrieval is poor | No separate fallback source — reflects and can regenerate or hedge |
| **Implementation** | Add a grading/filtering module to an existing pipeline | Requires the model itself to produce reflection judgments |
| **Best fit** | Existing pipelines where you want a bolt-on quality gate | Systems where you control or can fine-tune the generation model |

In practice, many production systems borrow from both without adopting either as a strict, named implementation: grading retrieved chunks before generation (the CRAG instinct) and asking the model to self-assess whether its answer is actually grounded before returning it (the Self-RAG instinct) are both patterns you can build incrementally on top of an existing RAG pipeline, even without a formal fallback-to-web-search step or a specially fine-tuned model.

---

## 23.6 Self-Awareness Inside the Loop vs. Evaluation Outside It

It's worth being precise about what problem this chapter is solving relative to Part VII. CRAG and Self-RAG build quality checks **into the RAG system's own execution loop** — they happen at query time, per request, and directly influence what gets generated or whether a correction is triggered.

Part VII's evaluation chapters (26-29) are about something related but distinct: measuring RAG system quality **from the outside**, across many queries, usually offline or in aggregate — retrieval metrics like recall and precision (Chapter 27), generation metrics like faithfulness (Chapter 28), and human-in-the-loop review (Chapter 29). That's how you know, in aggregate, whether your system is any good, and it's how you'd catch a systematic problem that any one query's self-grading might miss.

Think of it as the difference between a pilot's in-flight instrument checks (CRAG/Self-RAG — real-time, per-flight, self-generated) and an airline's post-flight safety audit program (Part VII — aggregate, external, statistically grounded). You need both. Per-query self-grading catches individual bad retrievals before they become bad answers; systematic offline evaluation catches patterns — like a retriever that's systematically weak on a whole category of question — that no single query's self-check would ever reveal.

---

## 23.7 The Honest Limitations

Teaching a system to grade itself is valuable, but it doesn't escape a fundamental problem: **the grader is still an LLM, subject to the same unreliability as the generator it's grading.**

- A relevance grader can itself be wrong — rating an actually-useful chunk as irrelevant (discarding good evidence) or rating a superficially-relevant chunk as good (missing exactly the trap it was built to catch)
- Self-RAG's "supported" judgment is the model checking its own work, and a model confidently wrong about a fact can just as easily be confidently wrong about whether that fact is supported
- Both approaches add **latency and cost** — grading is an extra LLM call (or extra tokens in the same call), and CRAG's fallback-to-web-search path adds a whole additional retrieval round trip when triggered
- CRAG's fallback source (commonly framed as web search in the original design) isn't always available or appropriate in an enterprise setting with private, closed knowledge bases — you may need a different fallback strategy, such as broadening the search or querying a secondary index
- Neither technique replaces systematic offline evaluation — self-grading catches individual bad retrievals, but only aggregate evaluation reveals whether your grader itself is well-calibrated

The honest framing: CRAG and Self-RAG reduce the rate of "confidently wrong from bad context" failures, in the same way RAG itself reduces (without eliminating) hallucination. They are a meaningful layer of defense, built with the same fallible material as everything else in the pipeline — not an escape hatch from imperfection.

---

## 23.8 Chapter Summary

- Not all retrieved context is equally good, and standard RAG pipelines generate from whatever was retrieved without checking its quality first.
- **Corrective RAG (CRAG)** adds an explicit grading step after retrieval, classifying chunks as relevant, ambiguous, or irrelevant, and takes corrective action — refining, supplementing, or discarding — before generation.
- **Self-RAG** trains the model to emit **reflection tokens** interleaved with generation: whether retrieval is needed at all, whether retrieved passages are relevant, and whether the generated answer is actually supported by them.
- Self-RAG's "is retrieval needed?" judgment avoids wasted retrieval on queries that don't need it; its "is this supported?" judgment is a direct, built-in check against hallucination despite grounding.
- These techniques build **self-awareness into the RAG loop itself**, operating per-query at execution time — a different layer from Part VII's systematic, aggregate, offline evaluation.
- Both approaches are still built from LLM judgments, which means the **grader can itself be wrong**, and both add latency and cost — they reduce bad-context failures, they don't eliminate them.

**Coming up next (Chapter 24):** we'll extend RAG beyond text entirely — retrieving and reasoning over images, tables, and video, and the multi-modal embedding and evaluation challenges that come with it.
