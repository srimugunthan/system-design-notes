# Chapter 19: Handling Contradictory/Noisy Context

## 19.1 What This Chapter Covers

So far we've mostly assumed that once the right documents are retrieved, the hard part is over. Real knowledge bases don't cooperate with that assumption. They accumulate an old policy document and its replacement, two team wikis that were never reconciled, a draft that was never marked as a draft, and a dozen other sources of quiet internal disagreement.

This chapter asks a question every production RAG system eventually runs into: **what happens when the context handed to the model doesn't agree with itself?** We'll look at why this happens, how a naive system fails silently when it does, and three complementary strategies for handling it — prioritizing sources by metadata, instructing the model to surface conflicts instead of resolving them invisibly, and pruning the problem out of the index before it ever reaches retrieval.

---

## 19.2 Why Knowledge Bases Contradict Themselves

It's tempting to imagine a company's knowledge base as a single, coherent, up-to-date source of truth. In practice, it's closer to an archive that keeps everything and forgets nothing. A few realistic ways contradiction creeps in:

- **Superseded versions.** A refund policy from 2023 says returns are accepted within 30 days. A 2026 update changed it to 14 days. Both documents are still sitting in the same document store, both get parsed, chunked, and embedded exactly the same way, and both are equally eligible for retrieval unless something tells the system otherwise.
- **Genuinely conflicting sources of truth.** Two internal teams each maintain their own policy page, written independently, and they simply disagree — not because one is outdated, but because nobody reconciled them in the first place.
- **Draft vs. final content.** A proposal document, a meeting note discussing a *possible* change, and the actual approved policy can all use similar language, and a retriever has no inherent way to know which one reflects reality.
- **Regional or tier-based variation that looks like contradiction.** "Refunds take 5 business days" and "refunds take 14 business days" might both be true — for different regions — and without the metadata to disambiguate, they look like a straight conflict.

None of this is a retrieval bug in the traditional sense. The retriever did its job: it found chunks that are semantically relevant to the query. The problem is that "relevant" and "correct" are not the same property, and a plain similarity search has no mechanism for telling them apart.

---

## 19.3 How Naive RAG Fails on Conflicting Context

Without any handling for this, a RAG system typically does one of two unhelpful things when it retrieves conflicting chunks:

1. **It silently picks one.** The model, forced to produce a single fluent answer, often just goes with whichever version happens to be phrased more confidently, appears earlier in the context (recall the lost-in-the-middle effects from Chapter 18), or simply "feels" more prominent statistically. The user has no idea a conflict existed — they just get an answer that might be the outdated one.
2. **It blends them into something that's wrong on its own terms.** Given "30 days" in one chunk and "14 days" in another, a model can produce a garbled synthesis — "returns within 14 to 30 days" — that isn't actually stated anywhere in either source. This is a particularly dangerous failure mode because it doesn't look like a hallucination in the classic sense (nothing was invented from nothing); it's a plausible-sounding blend of two real but incompatible facts, closely related to the "mixing up details" hallucination flavor described back in Chapter 1.

Both failure modes share the same root cause: the prompt handed the model two facts that disagree, and gave it no instruction for what to do about that. Left to its own devices, the model resolves the tension the way it resolves everything — by producing the most fluent-sounding continuation, not necessarily the most correct one.

> **Core idea:** a RAG system that retrieves conflicting sources and says nothing about the conflict isn't more accurate than one that surfaces it — it's just more confident-sounding while being wrong.

---

## 19.4 Strategy 1: Metadata-Based Source Prioritization

The most durable fix operates before the prompt is even assembled: give the retrieval and ranking layer enough metadata to know which source *should* win, rather than asking the generator to guess. This connects directly back to the metadata enrichment work described in Chapter 7 — this is one of the main payoffs of doing that work well.

Useful metadata fields for this purpose:

| Field | Purpose | Example |
|---|---|---|
| `effective_date` / `last_updated` | Prefer newer content over older | Prefer the 2026-01 policy over the 2023-06 one |
| `status` | Distinguish draft/proposal from approved content | Exclude or down-rank `status: draft` |
| `source_authority` | Rank canonical sources above secondary mentions | Prefer the official policy doc over a Slack export |
| `region` / `tier` / `scope` | Disambiguate content that's correct but conditional | Match the user's region to the applicable chunk |
| `superseded_by` | Explicitly link an old document to its replacement | Suppress the old version entirely, or label it as historical |

With this metadata in place, a few concrete techniques become available:

- **Recency boosting** — at retrieval or re-ranking time (Chapter 15), add a scoring bonus for more recent `effective_date` values so that, all else equal, current documents outrank stale ones.
- **Authority tiers** — treat certain sources (an official policy repository, a compliance-approved wiki) as higher-authority than others (a general Slack archive, informal notes), and reflect that in ranking rather than treating all retrieved text as equally trustworthy.
- **Hard filtering** — for some fields, filtering is safer than scoring. A document explicitly marked `status: superseded` often shouldn't be retrievable at all outside of an explicit "historical" query mode, rather than merely being down-ranked and still occasionally winning.

This is the cleanest fix because it resolves ambiguity using structured signal the system can actually trust, instead of asking a language model to infer document authority from the prose itself — which it generally cannot do reliably.

---

## 19.5 Strategy 2: Explicit Conflict-Surfacing Instructions

Metadata prioritization handles the cases where you *can* determine a clear winner. It doesn't handle the cases where two sources are both current, both authoritative, and simply disagree — the "two internal policies that were never reconciled" scenario. For that, the prompt itself needs to change.

Building on the grounding instructions from Chapter 17, add an explicit conflict-handling rule:

```
If the provided sources contain conflicting information on the
same point, do not silently choose one. Instead:
- State that the sources disagree.
- Briefly describe what each source says, with its citation.
- If one source is clearly more recent or more authoritative
  based on the provided metadata, you may note that — but still
  disclose the conflict.
```

The output this produces is meaningfully different from a naive answer:

> "Sources disagree on the return window: the 2023 policy [1] states 30 days, while the 2026 update [2] states 14 days. Based on the more recent effective date, 14 days is likely current — please confirm with the latest policy owner."

This is a strictly more honest answer than a flat "returns are accepted within 14 to 30 days," and it hands the disagreement to a human rather than resolving it invisibly. For high-stakes domains — the financial services context of Chapter 35 being a clear example — this kind of transparent hedging is often a hard requirement, not a nice-to-have.

---

## 19.6 Strategy 3: Deduplication and Version-Pruning at Index Time

The third strategy pushes the fix further upstream still: instead of managing conflict at retrieval or generation time, prevent stale duplicates from ever competing for a retrieval slot in the first place.

- **Version pruning.** When a new version of a document is ingested, actively retire the old version from the searchable index — either removing it outright or moving it to a clearly separate "archive" index that's excluded from default retrieval. This ties into the incremental indexing and freshness practices covered in Chapter 33.
- **Near-duplicate collapsing.** For content that's genuinely redundant rather than versioned (the same policy copy-pasted into three different wiki pages), collapsing near-duplicate chunks at indexing time — keeping one canonical copy and discarding or merging the rest — reduces both noise and the chance of an artificial "conflict" that isn't actually a conflict, just redundant phrasing.
- **Ownership and review workflows.** Some of this is genuinely not a technical problem. If two teams maintain conflicting policies, no amount of retrieval engineering resolves the underlying organizational disagreement — the index can only faithfully reflect ambiguity that hasn't been resolved by the humans who own the source documents. Surfacing the conflict (Strategy 2) is often what actually prompts that resolution to happen.

---

## 19.7 The Honest Limits of Conflict Handling

It's worth being clear-eyed about what these strategies do and don't achieve:

- **Metadata prioritization is only as good as the metadata.** If `effective_date` is missing, wrong, or inconsistently applied across a document set (a common real-world state of affairs), recency boosting silently does nothing or, worse, boosts the wrong document.
- **Conflict-surfacing instructions depend on the model correctly recognizing the conflict**, which requires that both conflicting chunks actually made it into the retrieved context in the first place. If only one version was retrieved, there's no conflict to surface — just a confidently wrong (or confidently right) answer with no way to tell which.
- **Version-pruning requires a reliable ingestion pipeline** that knows a new version supersedes an old one — which is not automatic. Two documents about the same policy don't announce their relationship to each other; someone (or some process) has to establish it.
- **Not all conflicts should be resolved by the system at all.** Some disagreements are genuinely organizational and shouldn't be silently ranked away — surfacing them honestly, even at the cost of a less tidy answer, is often the more responsible outcome.

---

## 19.8 Chapter Summary

- Real knowledge bases accumulate **outdated versions, unreconciled policies, and draft content** alongside current, authoritative material — retrieval relevance does not imply correctness.
- Naive RAG systems handle conflicting retrieved context poorly: they either **silently pick a source** or **blend conflicting facts into an answer that's stated nowhere**, a variant of the hallucination problem from Chapter 1.
- **Metadata-based prioritization** — recency, authority tiers, status flags, and hard filtering built on the enrichment work from Chapter 7 — resolves ambiguity using structured signal rather than asking the model to infer document authority from prose.
- **Explicit conflict-surfacing instructions** in the prompt push the model to disclose disagreement between sources rather than resolve it invisibly, producing more honest, verifiable answers.
- **Deduplication and version-pruning at index time** prevent stale or redundant content from ever competing for a retrieval slot, tying into the freshness practices of Chapter 33.
- None of these strategies fully substitutes for the others — good conflict handling combines metadata prioritization, prompt-level honesty, and clean indexing, and some conflicts are organizational problems that only humans can ultimately resolve.

**Coming up next (Chapter 20):** we'll shift from prose answers to structured, machine-parseable output — how to constrain generation to a schema, and how tool and function calling let the generation step do more than just write text.
