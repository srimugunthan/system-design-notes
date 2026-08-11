# Chapter 28: Generation Metrics

## 28.1 What This Chapter Covers

Suppose retrieval did its job perfectly — the exact right passage landed at rank 1, with a clean recall@k and nDCG score to prove it. Is the answer the model wrote from that passage actually good? Not necessarily. This chapter covers the metrics that answer a genuinely different question than the ones in Chapter 27: given the context the model *was* handed, did it use that context well? We'll cover **faithfulness** and **groundedness** (does every claim trace back to the source), **answer relevance** (does the answer actually address the question asked), and approaches to **hallucination detection**, along with an honest look at the central problem underlying most of these techniques: using an LLM to judge an LLM.

---

## 28.2 A Different Question Than Retrieval Metrics Ask

It's worth being precise about the boundary between Chapter 27 and this chapter, because conflating them is one of the most common mistakes in RAG evaluation (as flagged back in Chapter 26).

Retrieval metrics ask: *did we find the right material?* Generation metrics ask something that starts only *after* that material has already been found and handed to the model: *given exactly this context, did the model produce a good answer from it?*

This distinction matters because generation can fail in ways that have nothing to do with retrieval quality at all:

- The model might **ignore** the retrieved context and answer from parametric memory instead (Chapter 1), producing an answer that happens to be right, or happens to be wrong, for reasons unrelated to what it was given
- The model might **misread** the context — get a number wrong, flip a comparison, miss a negation ("the policy does *not* apply to...")
- The model might answer a **different question** than the one asked, drifting off-topic while still sounding fluent and confident
- The model might **blend** retrieved facts with memorized facts, producing an answer that's partially grounded and partially not, in a way that's hard to spot from the outside

Generation metrics exist to catch these failures specifically — independent of whether the retriever handed over good material in the first place. A useful mental frame: retrieval metrics grade the *inputs* to generation; generation metrics grade what the model did *with* those inputs.

---

## 28.3 Faithfulness and Groundedness

**Faithfulness** (sometimes called **groundedness**) asks: does every factual claim in the generated answer actually trace back to something present in the retrieved context?

> **Faithfulness** = the proportion of claims in an answer that are supported by the retrieved context, rather than invented or pulled from the model's own memorized (and potentially outdated or wrong) knowledge.

This is distinct from asking whether the answer is *true* in some absolute sense — a faithful answer could still be wrong if the retrieved context itself was wrong or outdated (a retrieval problem, not a generation problem). Faithfulness only asks whether the model stayed true to what it was given.

The dominant technique for measuring this, used by frameworks like RAGAS (Chapter 26), works in two steps:

1. **Claim decomposition** — break the generated answer down into individual, atomic factual claims. For example, the answer "Our premium plan costs $49/month and includes priority support" decomposes into two claims: (a) the premium plan costs $49/month, and (b) it includes priority support.
2. **Claim verification** — for each individual claim, check whether it's supported by the retrieved context. This check is typically done by prompting an LLM with the claim and the context and asking, essentially, "is this claim directly supported by this text — yes, no, or not addressed?"

The faithfulness score is then the fraction of claims that came back "supported." Let's work through a small illustrative example. Suppose an answer decomposes into 4 claims, and the verification step finds:

```
Claim 1: supported by context
Claim 2: supported by context
Claim 3: NOT supported (invented detail)
Claim 4: supported by context
```

Faithfulness = 3/4 = **0.75**. That single unsupported claim is worth flagging specifically, not just averaging away — in a production system, a human reviewing this output would want to know *which* claim was unsupported, not just that the aggregate score dipped, since one fabricated detail (say, a made-up dollar figure) can matter far more than the other three being correct.

Faithfulness is the closest thing this Part has to a direct measurement of hallucination, and it's why it's usually the first metric teams reach for when building a generation evaluation harness.

---

## 28.4 Answer Relevance

Faithfulness tells you whether an answer stuck to the source material. It says nothing about whether the answer actually addressed what the user asked. That's a separate metric: **answer relevance**.

> **Answer relevance** measures whether the generated answer actually addresses the question that was asked — independent of whether the answer is factually correct or faithful to the source.

It's entirely possible for an answer to be perfectly faithful (every claim traces cleanly back to the retrieved context) and still be a poor answer, because it dodges the actual question, buries the answer under irrelevant tangents pulled from the context, or answers a nearby-but-different question. Imagine a user asks "does our premium plan include phone support?" and the model responds with a faithful, fully-grounded paragraph describing the premium plan's pricing history — every sentence traceable to the source documents, and yet the user's actual question never gets answered.

A common technique for scoring answer relevance, again typically via LLM-as-judge: generate a small set of plausible questions that the *answer itself* seems to be answering, then measure how semantically similar those generated questions are to the original question the user actually asked. A high similarity suggests the answer is on-topic; a low similarity suggests drift. Some frameworks instead just prompt an LLM directly with the original question and the answer and ask it to rate relevance on a scale, with a rubric describing what "directly addresses the question" means versus "technically related but non-responsive."

Faithfulness and answer relevance are deliberately independent axes — a mature evaluation harness reports both, because a system optimizing for one without the other can drift into two different bad patterns: chasing faithfulness alone can produce answers that are safe, grounded, and unhelpfully vague; chasing relevance alone can produce answers that directly address the question but wander into unsupported claims to do so.

---

## 28.5 Hallucination Detection

We introduced hallucination back in Chapter 1 as a property of plain LLMs generating fluent but ungrounded text. In a RAG context, hallucination detection is really the *inverse framing* of faithfulness — instead of asking "what fraction of claims are supported," it asks "which specific claims are not, and how severe is that unsupported claim?"

The mechanics are largely the same claim-decomposition-and-verification pipeline described in Section 28.3, but hallucination detection typically goes one step further by categorizing the *kind* of unsupported claim, since not all hallucinations carry equal risk:

| Category | Description | Example |
|---|---|---|
| **Extrinsic hallucination** | Claim introduces information not present anywhere in the retrieved context | Context discusses a refund window; answer states a specific dollar refund amount never mentioned |
| **Intrinsic hallucination** | Claim contradicts or misstates something that *is* present in the context | Context says "refunds within 30 days"; answer says "refunds within 60 days" |
| **Unverifiable claim** | Claim isn't directly supported or contradicted — it's a reasonable inference, but not explicitly stated | Context describes a feature list; answer infers the product is "enterprise-grade" |

Intrinsic hallucinations (contradicting the source) are generally treated as more severe than extrinsic ones, since they represent the model actively misreading material it was given, rather than merely adding unsupported flourishes. Unverifiable claims are the hardest category to handle consistently — different reviewers, human or LLM, often disagree about where reasonable inference ends and unsupported speculation begins, and this ambiguity is worth designing your harness around rather than pretending it doesn't exist.

Once claims are categorized this way, a harness can compute not just an aggregate faithfulness score but a hallucination *rate* by severity — useful because "5% of claims are unverifiable inferences" and "5% of claims directly contradict the source" call for very different responses from an engineering team, even though they might otherwise be lumped into the same faithfulness number.

---

## 28.6 The Meta-Problem: Using an LLM to Judge an LLM

Nearly every technique in this chapter — claim decomposition, claim verification, relevance scoring — leans on an LLM to do the judging. It's worth being direct about what that costs you, because it's easy to treat an LLM-as-judge score as ground truth when it isn't.

**The judge can share the generator's blind spots.** If both the generator and the judge are built on similar underlying models, they may share systematic misunderstandings — a fact both models get wrong the same way, a nuance both models miss, a genre of phrasing both models find equally plausible-sounding. The judge isn't an independent check in the way a human with different training and different failure modes would be.

**The judge can be inconsistent.** Ask the same judge to score the same answer twice, and you can get slightly different scores — LLM outputs are not perfectly deterministic, and rubric-following has some noise built in. Averaging over larger test sets smooths this out somewhat, but on any single example, treat a judge score as an estimate, not a precise measurement.

**The judge has its own biases.** LLM judges have been observed to favor longer answers over shorter ones, to be swayed by confident phrasing regardless of accuracy, and to score more favorably when the answer's style resembles the judge's own preferred writing style. None of these biases track actual answer quality.

**The judge's prompt and model version matter more than people expect.** A small rewording of the judging rubric, or an upgrade to a newer version of the judge model, can shift scores meaningfully even when the system being evaluated hasn't changed at all — a point already raised in Chapter 26's discussion of harness stability. If you're comparing evaluation runs across time, pin your judge model version deliberately, the same way you'd pin any other dependency.

None of this means LLM-as-judge techniques are useless — far from it; they're what makes it feasible to evaluate thousands of generated answers on every pipeline change, something no team could do with human review alone. But it does mean these scores need periodic grounding against actual human judgment, rather than being trusted blindly forever. That calibration process — sampling judge-scored outputs and having a human check whether the judge got it right — is exactly what Chapter 29 covers.

---

## 28.7 A Note of Limitations

Beyond the LLM-as-judge meta-problem, a few more honest caveats about generation metrics specifically:

- **Claim decomposition is imperfect.** Breaking an answer into "atomic claims" is itself a judgment call, and different decompositions of the same answer can yield different faithfulness scores. A sentence with subtle compound meaning can be split inconsistently across runs.
- **Faithfulness doesn't guarantee correctness.** A perfectly faithful answer, grounded entirely in the retrieved context, is still wrong if the retrieved context itself is outdated or incorrect. Faithfulness measures fidelity to the source, not truth — those are only the same thing when your knowledge base itself is accurate and current, which is a separate concern addressed by data freshness practices in Chapter 33.
- **These metrics don't capture tone, style, or usefulness.** An answer can be faithful, relevant, and hallucination-free, and still be a poor user experience — too terse, too jargon-heavy, formatted badly for the channel it's delivered through. Automated generation metrics are necessary, not sufficient, for judging whether an answer is genuinely good for your actual users.

---

## 28.8 Chapter Summary

- Generation metrics answer a different question than retrieval metrics: given the context the model *was* handed, did it use that context well?
- **Faithfulness (groundedness)** measures what fraction of an answer's claims are supported by the retrieved context, typically via decomposing the answer into atomic claims and verifying each one against the source.
- **Answer relevance** measures whether the answer actually addresses the question asked, independent of whether it's factually correct or grounded — a faithful answer can still fail to be relevant.
- **Hallucination detection** is the inverse framing of faithfulness, often further categorized into extrinsic hallucinations (fabricated additions), intrinsic hallucinations (contradicting the source), and unverifiable inferences, since these carry different levels of risk.
- Most of these techniques rely on **LLM-as-judge** scoring, which introduces its own failure modes: shared blind spots with the generator, inconsistency between runs, systematic biases (like favoring longer or more confident-sounding answers), and sensitivity to judge model and prompt version.
- These metrics should be treated as **estimates that require periodic human calibration**, not as ground truth — the subject of Chapter 29.

**Coming up next (Chapter 29):** we close out Part VII by looking at how human judgment fits into this picture — golden datasets, annotation pipeline design, and how teams use human review to validate and calibrate the automated metrics from this chapter and the last.
