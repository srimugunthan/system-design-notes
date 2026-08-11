# Chapter 29: Human-in-the-Loop Evaluation

## 29.1 What This Chapter Covers

Every metric in the last two chapters — recall@k, MRR, nDCG, faithfulness, answer relevance — is computed automatically, which is exactly what makes them useful for running on every pipeline change. But automatic doesn't mean trustworthy by default, and it certainly doesn't mean complete. This chapter covers where human judgment still has to enter the picture: how to build and maintain a **golden dataset**, how to design an **annotation pipeline** that produces consistent and trustworthy labels, and how teams use human review to **calibrate** the automated metrics from Chapters 27 and 28, rather than trusting them blindly. This is also the closing chapter of Part VII, so we'll end by bridging into Part VIII, where the focus shifts from measuring quality to running a RAG system reliably in production.

---

## 29.2 Why Automated Metrics Aren't Enough On Their Own

Every automated metric covered so far has a quiet dependency baked into it. Recall@k, MRR, and nDCG (Chapter 27) all require a labeled set of known-relevant documents per query — that label came from somewhere, and "somewhere" is a human who read the query and the document and made a judgment call. Faithfulness and answer relevance (Chapter 28) are typically computed by an LLM judge — and Chapter 28 was explicit that an LLM judge has its own biases, inconsistencies, and blind spots that need to be checked against something more reliable.

That "something more reliable" is human review. Not because humans are infallible — they aren't, and we'll get to inter-annotator disagreement shortly — but because human judgment is the actual target these automated metrics are trying to approximate. An LLM-as-judge faithfulness score is only useful insofar as it *agrees with what a careful human reviewer would have concluded*. If it doesn't, the automated score isn't measuring what you think it's measuring, no matter how confidently it reports a number.

There's a second, distinct reason automated metrics aren't sufficient on their own: some quality dimensions are genuinely hard to fully automate, because they depend on context an LLM judge simply doesn't have. "Is this a good answer for our specific users" often means things like:

- Does it match the tone and reading level our actual customers expect?
- Does it correctly navigate an edge case specific to our business that a general-purpose judge model was never told about?
- Would a domain expert on our team consider this response complete, or technically correct but missing an important caveat they'd expect any competent person in the field to include?

An LLM judge can be prompted with a rubric that approximates these things, but a rubric written by an engineer is still a step removed from the actual standard your users and domain experts hold. Periodic human review closes that gap.

> **Key idea:** automated metrics are necessary because they're the only thing fast and cheap enough to run on every change — but they are a *proxy* for human judgment, not a replacement for it, and proxies need to be periodically checked against the thing they're standing in for.

---

## 29.3 What Is a Golden Dataset?

A **golden dataset** is a curated, trusted set of test questions, each paired with:

- A **verified correct answer** (or, for open-ended questions, a description of what a correct answer must contain)
- The **known-relevant source document(s)** that answer should be grounded in
- Ideally, **graded relevance judgments** for a broader set of documents, if you're computing nDCG (Chapter 27)

The word "golden" is doing real work here — it signals that this dataset has been reviewed and vetted by a human who trusts its labels, as opposed to a large but noisy sample of unreviewed production traffic. This is the labeled evaluation set that Chapter 27 flagged as a hard prerequisite for computing any retrieval metric at all, and it's also typically the basis for spot-checking generation metrics, since a golden answer gives a human reviewer something concrete to compare a generated answer against.

**Building a golden dataset** usually draws from a mix of sources:

1. **Real user queries**, sampled from production logs (or from a pilot/beta period before launch), because these best represent the actual distribution of what your system will be asked — not what engineers imagine users will ask
2. **Deliberately constructed edge cases**, written by the team to probe known-hard scenarios: ambiguous queries, multi-hop questions (Chapter 16), questions where the honest answer is "the documents don't say," questions that touch conflicting or outdated source material (Chapter 19)
3. **Domain expert review**, where someone who actually understands the subject matter — not just the engineering team — verifies that the "correct answer" is actually correct and that the "relevant documents" are actually the right ones

A golden dataset of even 100–200 well-chosen, carefully labeled questions is usually far more valuable than a much larger set of unreviewed ones, because the entire point is that you *trust* every label in it. A thousand noisy labels don't tell you as much as two hundred trustworthy ones — noise in your ground truth doesn't average out the way noise in a model's predictions does; it just quietly corrupts every metric computed against it.

**Maintaining a golden dataset** is ongoing work, not a one-time deliverable. It needs attention whenever:

- Your underlying documents change or get retired (Chapter 33 covers incremental indexing and freshness) — a golden answer grounded in a policy document that was superseded last quarter is now testing against a stale target
- Your product surface changes — new features mean new categories of questions users will actually ask
- You notice gaps — a category of production failure that your golden set never would have caught, which should be added as new labeled examples going forward

Treat the golden dataset the way you'd treat a software test suite: it grows and gets pruned over the life of the project, and letting it go stale silently undermines every metric run against it.

---

## 29.4 Designing an Annotation Pipeline

Producing labels — relevance judgments, correct answers, quality ratings — is itself a process that needs deliberate design, not an afterthought delegated to whoever's free that week.

**Who annotates.** There's a real tradeoff here. Subject-matter experts (a lawyer reviewing legal RAG answers, a support lead reviewing customer support answers) produce the most trustworthy labels but are the most expensive and hardest to schedule time with. General annotators or crowdworkers are cheaper and more available, but need clear guidelines and can't be relied on for judgments that require deep domain expertise — you would not want a general annotator judging whether a financial services answer (Chapter 35) correctly applies a regulatory nuance. Many teams use a hybrid: general annotators handle first-pass labeling against a well-written rubric, and domain experts review a sample or handle escalations where the rubric doesn't clearly resolve the case.

**Clear annotation guidelines.** A relevance judgment is only as consistent as the instructions behind it. "Is this document relevant to this query?" is more ambiguous than it sounds — relevant to answering it directly, or relevant as background context, or relevant to a plausible alternate interpretation of an ambiguous query? Good annotation guidelines spell this out with examples, including deliberately tricky borderline cases, the same way a good style guide resolves ambiguity for writers rather than leaving it to individual taste.

**Inter-annotator agreement.** If you have more than one annotator (and for anything you plan to trust, you should), have a subset of questions labeled independently by at least two people, and measure how often they agree. Low agreement is a diagnostic signal, not just noise to shrug off — it usually means one of two things: either the guidelines are ambiguous and need to be tightened, or the underlying question genuinely doesn't have a clean answer (which is itself useful to know, since if two careful humans can't agree, an LLM judge scoring the same case shouldn't be trusted to have gotten it "right" either). Tracking agreement over time also tells you whether your rubric is maturing — agreement should generally improve as guidelines get refined based on early disagreements.

**Sampling production traffic for review.** A golden dataset, however well built, is necessarily static and finite — it can't cover everything users will eventually ask. A complementary practice is to continuously sample a small percentage of real production queries and answers for human review, on a rolling basis. This serves two purposes: it surfaces failure categories your golden set doesn't yet cover (which then feeds back into growing the golden set, closing the loop described in Section 29.3), and it gives you a real-time read on production quality that a static test set, run only when you deploy a change, cannot provide on its own.

---

## 29.5 Using Human Review to Calibrate Automated Metrics

This is where the two halves of this Part come together. The automated metrics from Chapters 27 and 28 are what you run on every change, because they're fast. Human review is what you run periodically to check whether those automated metrics can still be trusted.

A practical calibration workflow looks something like this:

1. **Sample** a set of question/answer pairs that were already scored by your automated harness — say, 50 examples spanning a range of automated scores, from clearly high to clearly low to ambiguous middle-of-the-road cases
2. **Have humans independently score the same examples**, using the same quality dimensions (faithfulness, relevance, and so on) and the same or comparable rubric the LLM judge was given
3. **Compare human scores against the automated scores** for each example — where do they agree, and more importantly, where do they diverge?
4. **Diagnose the divergence.** Is the LLM judge systematically too lenient? Too harsh on a particular category of question? Fooled by confident-sounding but unsupported claims (a known bias flagged in Chapter 28)? Struggling specifically with multi-hop questions (Chapter 16) where "faithfulness" is harder to decompose cleanly?
5. **Act on what you find** — this might mean rewriting the judge's rubric prompt, swapping to a stronger judge model for certain question categories, adding a deterministic check for a failure mode the LLM judge keeps missing (echoing the custom-harness pattern from Chapter 26), or simply documenting that automated faithfulness scores need a wider margin of error on a specific question type

This calibration loop isn't a one-time setup step — it's a recurring practice, because judge models get upgraded, your product and document set evolve, and the categories of questions users ask shift over time. A judge that was well-calibrated against human review six months ago isn't guaranteed to still be well-calibrated today, especially after any change to the judge model or its prompt.

---

## 29.6 A Note of Limitations

Human-in-the-loop evaluation solves real problems, but it's worth being honest about its own limits before treating it as the final word:

- **Humans disagree too.** Inter-annotator agreement is rarely perfect, and treating a single human's judgment as unquestionable ground truth just relocates the calibration problem rather than solving it. What you're really building toward is *consensus among careful, well-guided reviewers* — not an infallible oracle.
- **Human review doesn't scale to every query.** It's precisely because human review is slow and expensive that automated metrics exist in the first place; human review is a calibration and spot-check mechanism, not a replacement for the automated harness in daily use.
- **Sampled review can miss rare but severe failures.** A production sampling process that reviews 1% of traffic will, by construction, miss most instances of a failure mode that only occurs in 1 out of 10,000 queries — and severity doesn't correlate with frequency. A rare hallucination in a financial services answer can matter more than a common minor phrasing issue.
- **Annotator fatigue and drift are real.** Reviewers doing repetitive labeling work over long stretches can become less careful or unconsciously shift their standards over time, which is another reason to periodically re-check agreement rather than trusting a labeling process indefinitely just because it was well-designed at the start.

None of this argues against building golden datasets and annotation pipelines — it argues for treating them, like the automated metrics they calibrate, as an ongoing practice that needs its own maintenance, rather than a project with a defined end date.

---

## 29.7 Chapter Summary

- Automated metrics (Chapters 27–28) are **necessary but not sufficient** — they are proxies for human judgment and need to be periodically validated against it, and some quality dimensions are inherently hard to fully automate.
- A **golden dataset** is a curated, trusted set of questions paired with verified correct answers and known-relevant documents, built from real production queries, deliberately constructed edge cases, and domain expert review.
- Golden datasets require **ongoing maintenance** as documents, products, and observed failure modes evolve — a stale golden set silently undermines every metric computed against it.
- **Annotation pipeline design** involves choosing who annotates (domain experts vs. general annotators, often a hybrid), writing clear guidelines, measuring **inter-annotator agreement**, and continuously **sampling production traffic** for review.
- Low inter-annotator agreement is a useful diagnostic signal — it points to either ambiguous guidelines or genuinely ambiguous underlying questions, either of which matters.
- **Calibrating automated LLM-as-judge metrics** against human review is a recurring workflow, not a one-time setup step, because judge models, products, and user behavior all change over time.
- Human-in-the-loop evaluation has its own limits — annotator disagreement, limited scale, and the risk of sampled review missing rare but severe failures — and needs maintenance just like the automated systems it calibrates.

**Coming up next (Chapter 30):** with Part VII's evaluation toolkit in hand — evaluation harnesses, retrieval metrics, generation metrics, and human calibration — we move into Part VIII and the practical work of running RAG systems in production: system design for scale, cost, monitoring, freshness, and security.
