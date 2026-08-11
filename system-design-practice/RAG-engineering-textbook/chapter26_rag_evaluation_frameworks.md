# Chapter 26: RAG Evaluation Frameworks

## 26.1 What This Chapter Covers

How do you know if your RAG system is actually any good?

Not "does it run without crashing" — that's a much lower bar. The real question is: when a user asks it something, does it find the right information, and does it answer with that information faithfully? This chapter introduces the tooling and mental model for answering that question systematically, rather than by eyeballing a handful of demo queries and calling it a day. We'll look at why RAG evaluation is structurally harder than evaluating a plain LLM, what a repeatable evaluation harness looks like, and how the major categories of evaluation frameworks — LLM-as-judge tools like RAGAS and TruLens, versus custom harnesses — approach the problem differently.

---

## 26.2 Why Evaluating RAG Is Harder Than Evaluating an LLM

If you're evaluating a plain LLM, you're really asking one question: *given this input, is the output good?* You can measure that with human ratings, benchmark suites, or automated scorers, and you're done — there's one system in the loop.

RAG breaks that simplicity apart. Recall the four-box pipeline from Chapter 1: retrieve, augment, generate. A RAG system's final answer is the product of **two separate subsystems** working together — a retriever that searches a knowledge base, and a generator that writes an answer from what it's given. Each subsystem can fail independently, and worse, their failures can hide each other.

Consider four ways a RAG answer can go wrong:

| Retrieval | Generation | Result |
|---|---|---|
| Found the right documents | Used them correctly | Good answer |
| Found the right documents | Ignored or misread them | Bad answer, but retrieval "worked" |
| Found the wrong documents | Answered anyway, ignoring them (used parametric memory) | Answer might *still* be correct, by luck |
| Found the wrong documents | Faithfully used the wrong documents | Confidently wrong answer |

That third row is the one that trips people up the most. A model with strong parametric knowledge (Chapter 1) can produce a correct-looking answer even when retrieval completely failed — because it already "knew" the answer from training and didn't actually need the retrieved context. If you only look at the final answer, you'd score this as a success and never notice the retriever is broken. The failure is masked until the day the model *doesn't* happen to know the answer, and by then it's in production.

The reverse also happens: retrieval does its job perfectly, hands over exactly the right passage, and the generator still botches the answer — misreading a number, missing a caveat, or contradicting the source outright. If you only look at whether retrieval found the right chunk, you'd conclude the system is fine, when the actual output the user sees is wrong.

> **Core idea:** A RAG system's end-to-end answer quality is not a single number you can measure once. You need to evaluate retrieval and generation *separately*, because each can succeed or fail independently, and only measuring the final output leaves you blind to which one is actually broken.

This is why Part VII devotes an entire chapter to retrieval metrics (Chapter 27) and a separate chapter to generation metrics (Chapter 28) — they answer genuinely different questions, and conflating them is one of the most common mistakes teams make when they first try to evaluate a RAG system.

---

## 26.3 The Idea of an Evaluation Harness

Before comparing specific tools, it helps to name the general shape of what all of them are trying to build: an **evaluation harness**.

> A RAG evaluation harness is a repeatable pipeline that runs a fixed set of test questions through your RAG system, captures what it retrieved and what it generated, and scores both against some standard — automatically, on demand, every time your system changes.

Think of it as the RAG equivalent of a test suite in software engineering. You wouldn't ship a code change without running your tests; you shouldn't ship a change to your chunking strategy, your embedding model, your prompt, or your re-ranker (all covered in later parts of this book) without running your evaluation harness. Without one, teams tend to fall back on "it looked fine when I tried three questions" — which is not evaluation, it's a vibe check, and vibe checks don't catch regressions.

A minimal harness has four ingredients:

1. **A test set of questions** — ideally paired with known-correct answers and known-relevant source documents (we'll come back to how these are built in Chapter 29)
2. **A way to run each question through the live pipeline** — capturing the retrieved chunks *and* the generated answer, not just the final text
3. **A set of scoring functions** — metrics applied to the retrieved chunks (Chapter 27) and metrics applied to the generated answer (Chapter 28)
4. **A report** — aggregated scores you can compare across runs, so you can tell whether a change made things better or worse

That fourth ingredient matters more than it sounds. A single evaluation run tells you almost nothing in isolation — "faithfulness is 0.81" means very little on its own. What matters is the *trend*: did faithfulness go up or down after you swapped embedding models, changed your chunk size, or rewrote your system prompt? A harness only earns its keep when it's run repeatedly and its outputs are compared over time.

---

## 26.4 Framework Category 1: LLM-as-Judge Frameworks

The first major category of evaluation tooling uses an LLM to grade another LLM's output. This sounds circular at first — and we'll be honest about its limits in a moment — but it solves a real practical problem: many quality dimensions in RAG (did the answer stay faithful to the source? is it actually relevant to the question?) are semantic judgments that simple string-matching metrics can't capture, and hiring humans to grade every test run is slow and expensive.

Two well-known examples of this category, at a conceptual level:

- **RAGAS** is an evaluation library purpose-built for RAG pipelines. It computes a small set of standardized metrics — including faithfulness and answer relevance, which Chapter 28 covers in depth — by prompting an LLM to decompose an answer into individual claims and check each one against the retrieved context. It's designed to plug into a harness with minimal setup: you feed it questions, retrieved contexts, and generated answers, and it returns scores.

- **TruLens** takes a broader "observability" angle. Rather than being narrowly scoped to RAG metrics, it wraps your whole application, logs every step of a pipeline run (what was retrieved, what prompt was constructed, what was generated), and lets you attach LLM-as-judge "feedback functions" to score any part of that trace. It's closer to an evaluation *and* tracing layer than a pure metrics library.

What both share is the underlying technique: instead of a human reading every answer, a capable LLM is given a rubric (a prompt describing what "faithful" or "relevant" means) and asked to score the output, often with a structured explanation of *why* it gave that score. This makes evaluation fast enough to run on every code change, which is exactly what a harness needs.

The tradeoff, stated plainly: an LLM judge is still an LLM, with all the failure modes from Chapter 1 — it can misjudge, it can be inconsistent between runs, and it can share blind spots with the model it's judging. We'll return to this honestly in Chapter 28, because it's a real limitation, not a footnote.

---

## 26.5 Framework Category 2: Custom Evaluation Harnesses

The second category is the one most production teams eventually build for themselves: a **custom harness**, purpose-built around their own domain, data, and definition of "correct."

Off-the-shelf frameworks like RAGAS and TruLens are excellent starting points, but they compute *general-purpose* metrics — faithfulness, relevance, recall — that don't know anything about your specific product. A custom harness fills that gap. Some concrete examples of things a general framework won't check for you:

- A legal RAG system might need to verify that every cited clause number actually exists in the retrieved contract, not just that the answer sounds faithful
- A customer support RAG system (Chapter 36) might need to check that the answer never promises a refund policy that doesn't exist in the company's actual policy documents
- A financial services RAG system (Chapter 35) might need every numeric figure in the answer cross-checked against the source with exact-match precision, because "approximately faithful" isn't good enough when the number is a dollar amount
- A code documentation RAG system (Chapter 37) might need to check that any code snippet in the answer actually compiles or matches the referenced API signature

Custom harnesses typically don't throw away the general metrics — they layer domain-specific checks *on top of* the standard retrieval and generation metrics from Chapters 27 and 28. A common pattern is:

1. Run the standard metrics (recall@k, faithfulness, etc.) via a library or hand-rolled scorer, for comparability across runs
2. Add domain-specific rule-based checks (regex, schema validation, numeric comparison) where correctness can be checked deterministically rather than by another LLM's judgment
3. Add domain-specific LLM-as-judge checks where the rubric is genuinely custom — "does this answer comply with our disclosure requirements?" is not a question RAGAS was built to answer out of the box

The general lesson: **off-the-shelf LLM-as-judge frameworks are a strong default, not a ceiling.** Use them to get an evaluation harness running quickly, then extend them with the checks that matter specifically to your product, the same way you would extend a general-purpose test framework with assertions specific to your application.

---

## 26.6 A Note of Honesty: Evaluation Is Not "Solved" by Buying a Tool

It's tempting to treat "we installed RAGAS" as equivalent to "we have RAG evaluation." It isn't. A framework gives you the *mechanism* for scoring — it does not give you a good test set, it does not tell you what threshold counts as "good enough" for your use case, and it does not replace the judgment of someone who understands your users.

A few honest caveats worth internalizing before you build a harness:

- **Metrics without a test set are meaningless.** Every metric in this Part assumes you have questions with known-relevant documents and, in many cases, known-correct answers. Building and maintaining that set is real, ongoing work — covered in Chapter 29 — not a one-time task.
- **A high aggregate score can hide bad tail behavior.** A harness that reports "faithfulness: 0.93" across a hundred questions can still be badly wrong on the handful of questions that matter most to a specific user, especially if your test set doesn't represent the actual distribution of production traffic.
- **LLM-as-judge scores drift.** Swap the judge model, change its prompt slightly, or upgrade its version, and your scores can shift even though your RAG system didn't change at all. Treat judge-model changes as a variable you need to control for, not a free upgrade.
- **No framework catches what it wasn't designed to check.** General frameworks catch general failure modes. They will not catch that your customer support bot just promised something your company doesn't actually offer, unless you specifically built a check for it.

None of this is a reason to avoid these tools — it's a reason to treat "evaluation framework installed" as the starting line, not the finish line.

---

## 26.7 Chapter Summary

- RAG systems have **two subsystems that can fail independently** — retrieval and generation — and failures in one can mask or compound failures in the other, so end-to-end scores alone are not enough.
- A **RAG evaluation harness** is a repeatable pipeline: a fixed test set, a way to capture retrieved chunks and generated answers, scoring functions, and a report you can compare across runs.
- **LLM-as-judge frameworks** like RAGAS and TruLens use an LLM to score qualities like faithfulness and relevance by decomposing answers and checking them against source context, making evaluation fast enough to run on every change.
- **Custom harnesses** layer domain-specific, often deterministic checks on top of general metrics, because off-the-shelf frameworks don't know your product's specific correctness requirements.
- Evaluation tooling is a **mechanism, not a solution** — it still requires a good test set, sensible thresholds, and awareness that LLM judges have their own biases and drift.
- Chapters 27 and 28 will go deep on the two families of metrics these frameworks compute: **retrieval metrics** (did we find the right documents) and **generation metrics** (did the model use them correctly).

**Coming up next (Chapter 27):** we'll zoom into the retrieval side of evaluation and work through recall@k, MRR, and nDCG with concrete worked examples, and see exactly what kind of labeled data you need to compute them.
