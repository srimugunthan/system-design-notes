# Chapter 27: Retrieval Metrics

## 27.1 What This Chapter Covers

If the retriever hands the generator the wrong documents, nothing downstream can save the answer — the model is being asked to write a grounded response from ungrounded material. So before we can trust anything the generator produces, we need a way to measure, in isolation, whether retrieval is doing its job. This chapter covers the three metrics you'll see most often for that purpose: **recall@k**, **MRR (Mean Reciprocal Rank)**, and **nDCG (Normalized Discounted Cumulative Gain)** — what each one actually measures, how to compute it by hand on a small example, and why none of them can be computed without a specific kind of labeled data.

---

## 27.2 What Are We Actually Measuring?

A retriever, given a query, returns a ranked list of results — typically the top *k* chunks it judged most relevant, where *k* might be 5, 10, or 20 depending on your system. Retrieval evaluation asks a deceptively simple question: **how good is that ranked list?**

But "good" can mean different things depending on what you care about:

- Did the relevant document show up *anywhere* in the list at all?
- If it showed up, was it near the *top*, or buried at the bottom?
- If *multiple* documents were relevant, and some more relevant than others, did the best ones get ranked highest?

Each of these is a different question, and each has its own metric. Recall@k answers the first, MRR the second, and nDCG the third (and most nuanced). We'll take them in that order, building up complexity as we go.

---

## 27.3 Recall@k: Did We Find It At All?

**Recall@k** asks the most basic question you can ask of a retriever: among the top *k* results it returned, did at least one relevant document appear?

> **Recall@k** = (number of relevant documents found in the top k results) ÷ (total number of relevant documents that exist for that query)

Let's work a small illustrative example. Suppose for a given query there are 2 documents in your knowledge base that are genuinely relevant — call them D1 and D7. Your retriever returns the top 5 results:

```
Rank 1: D3
Rank 2: D1   ← relevant
Rank 3: D9
Rank 4: D12
Rank 5: D4
```

Only D1 was found; D7 never showed up in the top 5. Recall@5 = 1/2 = **0.5**.

Now suppose a different query has only one relevant document, D1, and it appears at rank 2 of the top 5. Recall@5 = 1/1 = **1.0** — recall@k doesn't care *where* in the top k the document landed, only *whether* it landed there. That's both its strength (simple, easy to reason about) and its weakness (it can't distinguish between a relevant document at rank 1 and the same document buried at rank 5).

You'll often see recall reported at multiple values of *k* — recall@3, recall@10, recall@20 — because the right value of *k* depends on how many chunks you actually pass to the generator. If your system only sends the top 5 chunks into the prompt (Chapter 18 covers context window management), then recall@20 is close to meaningless for judging real-world performance — a relevant document at rank 15 will never reach the model anyway. Measure recall at the *k* your pipeline actually uses.

A related and commonly reported variant is **precision@k**, the mirror image: of the *k* results returned, what fraction were actually relevant? A retriever that returns 1 relevant chunk and 9 irrelevant ones alongside it has high recall (if that 1 chunk was the only relevant one) but poor precision — and poor precision has real cost, since irrelevant chunks still consume context window space and can distract the generator (Chapter 19 covers noisy context in more depth).

---

## 27.4 MRR: How Quickly Did We Find the First Good One?

Recall@k tells you *whether* a relevant document appeared. **Mean Reciprocal Rank (MRR)** tells you *how high up* the first relevant result appeared — which matters a great deal in practice, since most RAG pipelines weight or truncate context, giving earlier-ranked chunks more influence on the final answer.

For a single query, the **reciprocal rank** is simply 1 divided by the rank position of the first relevant result:

> **Reciprocal Rank** = 1 ÷ (rank of the first relevant result)

If the first relevant document appears at rank 1, reciprocal rank = 1/1 = 1.0 (perfect). If it appears at rank 4, reciprocal rank = 1/4 = 0.25. If no relevant document appears at all in the returned list, reciprocal rank = 0.

**MRR** is just the average of this value across a whole test set of queries. Let's compute it for three illustrative queries:

| Query | Rank of first relevant result | Reciprocal Rank |
|---|---|---|
| Q1 | 1 | 1.00 |
| Q2 | 3 | 0.33 |
| Q3 | 2 | 0.50 |

MRR = (1.00 + 0.33 + 0.50) / 3 = **0.61**

A few things worth noticing about MRR. First, it only cares about the *first* relevant result — if Q1 also had a second relevant document at rank 8, MRR wouldn't reflect that at all. That makes MRR well suited to queries with a single clear correct answer (think: "what is our current refund window?" — there's one authoritative policy document), and less suited to queries where multiple documents are genuinely relevant and you want credit for surfacing several of them well. Second, MRR punishes low ranks harshly and non-linearly: going from rank 1 to rank 2 costs you 0.5 points, but going from rank 9 to rank 10 barely moves the number at all. This mirrors real user behavior reasonably well — people notice the difference between "first result" and "second result" far more than between "ninth" and "tenth."

---

## 27.5 nDCG: Rewarding the Right Order, With Graded Relevance

Recall@k and MRR both treat relevance as binary — a document either is or isn't relevant. But in practice, relevance is often a matter of degree. For the query "what is our late payment policy," one document might be *the* authoritative policy page (highly relevant), another might mention late payments only in passing as part of a broader FAQ (somewhat relevant), and a third might be about a completely different policy (not relevant at all). **nDCG (Normalized Discounted Cumulative Gain)** is built to handle exactly this — graded relevance, and it rewards ranking the *most* relevant documents highest, not just *any* relevant document highest.

nDCG is built from two ideas stacked together:

**Discounted Cumulative Gain (DCG)** sums up the relevance scores of your returned results, but "discounts" (reduces the value of) documents that appear lower in the ranking — because a user is less likely to read that far down. A common formulation divides each document's relevance score by the logarithm of its rank position, so lower ranks contribute less.

**Normalization** compares your actual DCG against the *ideal* DCG — the DCG you'd get if the results were sorted in the perfect order, most relevant first. Dividing actual DCG by ideal DCG gives you a score between 0 and 1, where 1.0 means your ranking was already perfect.

Let's work through a small illustrative example. Say relevance is graded 0 to 2 (0 = not relevant, 1 = somewhat relevant, 2 = highly relevant), and your retriever returns 3 results with these graded relevance scores in this order:

```
Rank 1: relevance = 1
Rank 2: relevance = 2
Rank 3: relevance = 0
```

The highly relevant document (score 2) is at rank 2, not rank 1 — a real ranking mistake, even though every retrieved document that's relevant was technically "found." nDCG will penalize this ordering, whereas recall@k would not have noticed anything wrong at all (both relevant documents were found in the top 3 either way). This is exactly the gap nDCG exists to close: it's sensitive to order, and it's sensitive to *how* relevant each result is, not just whether it clears a relevant/not-relevant threshold. The ideal ordering here would have been [2, 1, 0] — most relevant first — and nDCG measures how far your actual ordering [1, 2, 0] falls short of that ideal.

Because nDCG requires *graded* relevance labels rather than simple binary ones, it's more expensive to set up than recall@k or MRR — someone has to decide not just "is this document relevant" but "how relevant, on a scale." That extra labeling cost is usually worth it for production systems where getting the *best* document to the top actually changes what a user or a generator sees first, but it's a real cost, and many teams start with recall@k and MRR and only add nDCG once they have a mature enough labeling process to support it.

---

## 27.6 Comparing the Three Metrics

| Metric | Question it answers | Relevance type | Sensitive to rank order? | Typical use |
|---|---|---|---|---|
| **Recall@k** | Did we find the relevant document(s) at all, within the top k? | Binary | No (within top k) | Quick health check, easy to compute and explain |
| **MRR** | How high did the *first* relevant result rank? | Binary | Yes (position of first hit) | Single-answer queries, e.g. factual lookups |
| **nDCG** | Did we rank the *most* relevant results highest, in the right order? | Graded | Yes (full ranking, weighted) | Nuanced ranking quality, multi-document relevance |

In practice, mature RAG evaluation harnesses (Chapter 26) report more than one of these side by side, because they catch different failure modes. A retriever can have excellent recall@10 (it eventually finds everything relevant) but poor MRR (the good stuff is buried at rank 8), and that gap tells you your retriever's *search* is fine but its *ranking* needs work — pointing you toward re-ranking (Chapter 15) rather than, say, a fundamentally different embedding model (Chapter 9).

---

## 27.7 The Prerequisite Nobody Can Skip: A Labeled Evaluation Set

Every metric in this chapter shares the same hard dependency: you cannot compute any of them without knowing, in advance, which documents are actually relevant to each test query. Recall@k needs to know the total count of relevant documents. MRR needs to know which rank the *first* relevant one landed at. nDCG needs graded relevance judgments for every retrieved result.

This labeled set — a collection of test queries, each paired with the documents that are genuinely relevant to it (and, for nDCG, *how* relevant) — doesn't appear for free. It has to be built, typically by someone who understands the domain well enough to judge relevance correctly, and it has to be maintained as your document collection changes, since documents get added, updated, or retired over time.

> **Key idea:** retrieval metrics don't measure retrieval quality in the abstract — they measure how well your retriever's output matches a set of human relevance judgments. The metrics are only as trustworthy as the labels underneath them.

This is precisely the work Chapter 29 covers in depth: how to design an annotation pipeline, how to get consistent relevance judgments from human reviewers, and how to keep a golden dataset healthy as your system and your underlying content evolve. Nothing in this chapter works without it — treat the labeled set as the foundation, not an afterthought.

---

## 27.8 A Note of Limitations

These metrics measure whether the *right chunks* were retrieved — they say nothing about what happens next. A retriever can score perfectly on recall@k, MRR, and nDCG, and the generator can still misread the retrieved text, contradict it, or ignore it entirely. Retrieval metrics are necessary but not remotely sufficient for judging end-to-end answer quality — that's the job of the generation metrics in Chapter 28.

There's also a subtler limitation worth naming: these metrics assume "relevant" is a stable, well-defined property of a document with respect to a query. In reality, relevance can be genuinely ambiguous, context-dependent, or even a matter of legitimate disagreement between two equally qualified human annotators. A document might be highly relevant to one user's intent behind a query and irrelevant to another user's different intent behind the *same* query text. No amount of metric sophistication fixes an underlying label that two reasonable people would have assigned differently — which is exactly why inter-annotator agreement, covered in Chapter 29, is something you need to actively check rather than assume.

---

## 27.9 Chapter Summary

- Retrieval evaluation requires measuring the ranked list a retriever returns against a set of known-relevant documents for each query.
- **Recall@k** measures whether relevant documents were found anywhere in the top k results — simple, but blind to ranking order.
- **MRR (Mean Reciprocal Rank)** measures how high the *first* relevant result ranked, averaged across queries — well suited to single-answer queries, punishes low ranks non-linearly.
- **nDCG (Normalized Discounted Cumulative Gain)** measures whether the *most* relevant results were ranked highest, using graded (not just binary) relevance, and is the most nuanced but most labeling-intensive of the three.
- Different metrics catch different failure modes — a system can score well on one and poorly on another, and that gap is diagnostically useful.
- **None of these metrics can be computed without a labeled evaluation set** — test queries paired with known relevant (and ideally graded) documents — which must be deliberately built and maintained.
- Retrieval metrics tell you nothing about generation quality; a perfect retrieval score can still be followed by a badly generated answer.

**Coming up next (Chapter 28):** we shift from measuring what was *found* to measuring what was *written* — faithfulness, answer relevance, groundedness, and how to detect hallucination even when retrieval did everything right.
