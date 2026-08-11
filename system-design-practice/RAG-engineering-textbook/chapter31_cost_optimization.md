# Chapter 31: Cost Optimization

## 31.1 What This Chapter Covers

Chapter 30 asked "how fast can this system respond?" This chapter asks the question that eventually lands on someone's desk in a budget review: **where is the money actually going, and which of it is worth spending?**

RAG systems have a cost structure that's easy to misjudge if you've only ever budgeted for a plain chatbot. There's ingestion-time cost, storage cost, query-time retrieval cost, and generation cost — and they don't scale the same way with traffic. Getting this wrong either means an unpleasant cloud bill or, just as often, over-aggressive cost cutting that quietly degrades answer quality. This chapter is about finding the levers that save real money without doing that.

---

## 31.2 Where the Money Goes

A useful way to think about RAG cost is to split it by *when* it's incurred and *how it scales*.

| Cost source | When incurred | Scales with | Typical share |
|---|---|---|---|
| Embedding (ingestion) | Once per document, plus re-indexing | Corpus size | Small, one-time-ish |
| Embedding (query time) | Every request | Query volume | Small per-request |
| Vector DB storage & compute | Continuously | Corpus size × dimensionality | Moderate, steady |
| Re-ranking | Every request (per candidate) | Query volume × candidates | Small to moderate |
| LLM generation tokens | Every request | Query volume × context size × output length | **Usually the largest and most variable** |

The headline fact, mirroring Chapter 30's latency finding: **generation is usually both the largest cost line and the most variable one.** Embedding and storage costs grow with the size of your corpus, which changes slowly. Generation cost grows with traffic and with how much context you stuff into every single prompt — both of which can spike unpredictably. A quiet product launch that suddenly gets attention can 10x your token spend overnight without touching your corpus at all.

This asymmetry matters for where you spend your optimization effort: shaving 20% off embedding cost is a nice one-time win; shaving 20% off tokens-per-request is a recurring, compounding win that scales with every future request.

---

## 31.3 Embedding Cost

Embedding cost has two components worth separating.

**Ingestion-time embedding** happens once per chunk (Chapter 6) when a document enters the system, and again whenever a document changes and needs re-embedding (Chapter 33). For a large, mostly-static corpus, this is a bounded, predictable cost. It becomes a *recurring* cost problem mainly when the corpus churns quickly or when you re-embed the entire corpus after every embedding model upgrade — a decision worth making deliberately, not automatically.

**Query-time embedding** happens on every single request, since the query itself needs to become a vector before search can happen. This cost is small per request but multiplies by traffic volume, so at scale it's worth the same scrutiny as any other per-request cost.

Levers for both:

- **Choose the smallest embedding model that meets your quality bar.** Chapter 9 covers the quality tradeoffs in depth; from a pure cost lens, a smaller model is cheaper to run and often produces smaller vectors, which also reduces storage and search cost downstream. Don't reach for the largest available embedding model by default — benchmark whether the retrieval quality difference actually matters for your use case.
- **Cache embeddings aggressively.** As covered in Chapter 30, embeddings are deterministic for a given model version, making them one of the easiest and safest things in the whole pipeline to cache.
- **Batch embedding calls at ingestion time.** Most embedding APIs and self-hosted models are meaningfully more cost-efficient per item when called in batches rather than one document at a time.

---

## 31.4 Retrieval and Storage Cost

Vector database cost is a function of how much you store and how much compute you spend searching it. A few practical levers:

- **Reduce vector dimensionality where quality allows.** Some embedding models offer smaller output dimensions (or support dimensionality reduction techniques) with only a modest quality cost. Lower dimensions mean less storage and faster search.
- **Tier your storage.** Not all content needs to live in the fastest, most expensive index tier. Infrequently queried archival content can sit in a cheaper storage class, with the hot, frequently accessed portion of the corpus in the higher-performance tier (Chapter 10 and Chapter 11 cover the indexing tradeoffs behind this).
- **Prune the corpus.** RAG cost optimization is sometimes less about clever engineering and more about housekeeping: stale, duplicate, or superseded documents that were never removed just sit there costing storage and occasionally getting retrieved incorrectly. Chapter 33's incremental indexing discipline pays a cost dividend here too.
- **Re-ranking cost scales with candidate pool size.** A re-ranker that scores 100 candidates costs roughly 10x what one scoring 10 candidates does. Retrieving a smaller, better first-pass candidate set (a good retriever, not just a big one) is often cheaper than compensating with a large candidate pool and heavy re-ranking.

---

## 31.5 Token Cost — The Big One

This is where most of the optimization effort belongs, because it's usually the largest and most controllable line item.

**Reduce retrieved context size.** Chapter 18 covers context window management in depth, but the cost angle is simple: every chunk you stuff into the prompt is tokens you pay for, on every single request, whether or not the model actually needed it. Retrieving 20 chunks "just in case" when 5 well-chosen ones would answer the question is a direct, recurring cost tax. Tightening your retrieval and re-ranking to return fewer, higher-precision chunks is simultaneously a quality lever and a cost lever — one of the rare cases where the two point the same direction.

**Model routing.** Not every query needs your most capable, most expensive model. A **model router** classifies incoming queries (or their retrieved context) by difficulty and sends easy, well-supported questions to a smaller, cheaper model, reserving the expensive model for queries that are genuinely hard, ambiguous, or high-stakes.

```
Query ──► Difficulty classifier
              │
     ┌────────┴────────┐
     ▼                  ▼
 "simple, well-       "ambiguous, multi-hop,
  grounded lookup"      or low retrieval confidence"
     │                  │
     ▼                  ▼
 Cheap/fast model    Strong/expensive model
```

The classifier itself can be lightweight — retrieval confidence scores, query complexity heuristics, or even a small cheap model making the routing call are all reasonable starting points. The savings compound because, in most real query distributions, a large share of traffic is genuinely simple.

**Batching generation where latency allows.** For asynchronous or non-interactive use cases (bulk document summarization, offline report generation), batching multiple generation requests can reduce per-token cost compared to always paying for real-time, low-latency single requests. This tradeoff only works where Chapter 30's latency budget isn't in play — batch what can tolerate delay, and keep the interactive path fast.

**Prompt compression and shorter instructions.** System prompts and few-shot examples (Chapter 17) are paid on every request too. A verbose, unmaintained system prompt that's grown over months of patches is a quiet, recurring cost — worth periodically auditing and trimming just like any other part of the context.

> **Core idea:** because token cost scales with both traffic and per-request context size, the highest-leverage cost optimization is usually the same lever that improves precision — retrieve less, but retrieve the right things.

---

## 31.6 A Note of Honesty — Cost Cutting Has a Quality Floor

Every lever in this chapter has a quality cost if pushed too far, and it's worth being explicit about that rather than presenting cost optimization as free money.

- **Smaller embedding models can quietly degrade retrieval quality**, which shows up not as an obvious failure but as a slow drift toward slightly-worse-than-before answers — hard to notice without the retrieval metrics from Chapter 27.
- **Aggressive context trimming risks losing the one chunk that actually answered the question.** Fewer tokens is cheaper, but "fewer" and "sufficient" are not the same target, and tuning too hard toward the former erodes the latter.
- **Model routing is only as good as its classifier.** A router that misjudges a hard query as easy sends it to a model that isn't equipped to handle it, and the user experiences this as a wrong answer, not as a cost-saving success.
- **Caching (Chapter 30) and cost optimization pull in the same direction most of the time, but not always** — a semantic cache tuned loosely to save more money is also a semantic cache more likely to return a mismatched answer.

The discipline that keeps all of this honest is measurement: track cost per query alongside the retrieval and generation quality metrics from Part VII, and treat any cost-saving change as a hypothesis to validate, not a free win to assume.

---

## 31.7 Chapter Summary

- RAG cost splits across embedding (ingestion and query time), vector storage/compute, re-ranking, and generation tokens — and **generation tokens are usually the largest and most volatile cost**, scaling with both traffic and context size.
- **Embedding cost** is reduced by choosing the smallest adequate model, caching aggressively, and batching ingestion-time calls.
- **Storage and retrieval cost** benefit from lower-dimensional vectors where quality allows, storage tiering, and pruning stale content from the corpus.
- **Token cost**, the biggest lever, is best addressed by reducing retrieved context size, **model routing** between cheap and expensive models based on query difficulty, batching non-interactive workloads, and trimming bloated prompts.
- Reducing retrieved context size is one of the few optimizations that improves **both** cost and precision simultaneously.
- Every cost lever has a quality floor — smaller models, tighter context, and routing all carry real risk of silent quality degradation if pushed without measurement.
- Cost optimization should be paired with the retrieval and generation metrics from Part VII, not treated as a standalone exercise.

**Coming up next (Chapter 32):** with latency and cost under control, the next question is how you'd even know if quality started slipping in production — this is where monitoring, retrieval tracing, and feedback loops come in.
