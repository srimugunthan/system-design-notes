# Chapter 33: Incremental Indexing & Data Freshness

## 33.1 What This Chapter Covers

Go back to Chapter 1's core promise: RAG solves the knowledge-cutoff problem because you can just "update the knowledge base" instead of retraining the model. That sentence is true, but it hides an operational question this chapter has to answer honestly: **update it how, exactly, and how often?**

For a small, mostly-static knowledge base, the answer is almost trivial — re-embed everything, rebuild the index, done. For a knowledge base backed by a live wiki, a ticketing system, or a database that changes thousands of times a day, "rebuild everything" stops being an option. This chapter is about **incremental indexing**: detecting what actually changed, updating only that, and keeping the index honestly in sync with fast-moving source systems.

---

## 33.2 Why Full Re-indexing Doesn't Scale

Rebuilding an index from scratch means re-parsing (Chapter 5), re-chunking (Chapter 6), re-embedding (Chapter 9), and re-inserting every single document, even the 99% that haven't changed since yesterday. For a small corpus this is cheap and simple, and honestly, if that describes your system, you should seriously consider just doing it — full rebuilds are dramatically easier to reason about than incremental updates, and simplicity has real value.

But full re-indexing breaks down along two axes as a system grows:

- **Cost.** Re-embedding an entire corpus on every update, when only a handful of documents actually changed, wastes the embedding budget from Chapter 31 on work that produces identical output to what you already had.
- **Freshness lag.** If a full rebuild takes hours, and you only run it once a day, then a document that changed at 9am isn't reflected in answers until the next day's rebuild — even though the whole point of RAG, per Chapter 1, was to avoid exactly this kind of staleness.

The fix is to stop treating indexing as an all-or-nothing batch job and start treating it as an ongoing sync process: detect what changed, and touch only that.

---

## 33.3 What Incremental Indexing Requires

Incremental indexing has three jobs, and it's worth naming them separately because each one fails in a different way if skipped.

**1. Detecting changes.** You need a reliable way to know that a source document has been added, modified, or removed since the last sync. Common approaches:

| Approach | How it works | Tradeoff |
|---|---|---|
| Change timestamps | Source system exposes `last_modified`; poll and compare | Simple, but misses deletes unless the system also reports them |
| Content hashing | Hash each document's content; compare hash to last known value | Catches silent edits even without reliable timestamps, at the cost of re-hashing on every sync |
| Change data capture (CDC) / webhooks | Source system pushes an event on every change | Near real-time, but requires the source system to support it |
| Full diff against a manifest | Periodically compare the full current document list to a stored manifest | Reliable catch-all for detecting deletes, but heavier to run |

Most production systems end up combining these: event-driven updates for the common case, plus a periodic full-diff reconciliation pass as a safety net, because event streams occasionally drop or duplicate messages, and nobody wants "the index quietly diverged from reality three weeks ago" to be the diagnosis.

**2. Updating only the affected chunks.** A single changed document might map to many chunks (Chapter 6). When that document changes, you don't need to touch unrelated documents — but you do need to correctly identify and replace *all* of that document's chunks, not just re-embed one and leave stale siblings behind. This requires keeping a stable mapping from source document to its set of chunk IDs, so an update can cleanly say "delete these N old chunks, insert these M new ones."

**3. Avoiding orphaned vectors.** This is the failure mode that's easiest to overlook and most damaging when missed: a document is deleted from the source system, but nobody tells the vector index, so its chunks stay searchable forever. The retriever keeps happily returning content that, as far as the source of truth is concerned, no longer exists. This is worse than simple staleness — it's actively misleading, because nothing about the retrieval result signals that the content has been retracted.

> **Core idea:** an index that only knows how to *add* is not incremental indexing — it's accumulation. Real incremental indexing must handle updates and deletes as first-class operations, not as things that get cleaned up "eventually."

---

## 33.4 Versioning Strategies

Once updates and deletes are handled mechanically, a second problem shows up: what happens when a retrieval query, mid-flight, spans an index that's half-updated? Or when two versions of a policy document both technically still exist in the corpus for a short window?

**Document version and timestamp metadata.** Every chunk should carry metadata identifying which version of its source document it came from and when that version became effective — not just when the chunk was created. This is the same metadata discipline from Chapter 7, applied specifically to freshness. At minimum, track:

- A source document ID stable across versions
- A version number or content hash for the specific revision
- An effective/modified timestamp
- A superseded flag or superseded-by pointer, where applicable

**Why this matters for conflict resolution.** Chapter 19 covered how a generation step should handle contradictory retrieved context — for example, preferring the more recent of two conflicting chunks. That strategy only works if the underlying metadata actually carries a reliable timestamp. Incremental indexing is the upstream half of that story: if stale chunks aren't marked as superseded (or aren't removed at all), Chapter 19's conflict-resolution logic has nothing reliable to reason about, and the model is left guessing which of two contradictory chunks to trust.

**Atomic-enough swaps.** For anything beyond a single-chunk update, aim to make the delete-old/insert-new operation as close to atomic as your vector database allows, so a query never observes a state with the new chunks present and the old ones not yet removed (duplicated, conflicting content) or vice versa (a gap where neither is available). Most vector databases don't offer full transactional guarantees across a batch of operations, so this is often approximated — briefly tolerating either duplication or a gap, and choosing deliberately which of the two failure modes is safer for your use case, is a more realistic goal than assuming true atomicity.

---

## 33.5 Keeping Pace with Fast-Moving Source Systems

The operational challenge compounds when the knowledge base draws from multiple source systems that change at very different rates — a rarely-updated policy PDF repository alongside a support ticket system that changes every few seconds.

A few practical patterns:

- **Tier sync frequency by source volatility.** Not every source needs the same freshness SLA. A ticketing system might sync near-real-time via webhooks; a policy document archive might sync nightly. Applying a single sync cadence to all sources either wastes resources on the slow-moving ones or under-serves the fast-moving ones.
- **Backpressure and batching for bursty sources.** A source system that emits thousands of change events in a short burst (a bulk import, a mass re-tagging) shouldn't be allowed to trigger thousands of individual embedding calls in real time — batch and queue these to protect both cost (Chapter 31) and downstream index stability.
- **Monitor sync lag as a first-class metric.** "How far behind is the index from the source system, right now?" deserves its own dashboard, feeding into the drift detection story from Chapter 32 — a growing lag is an early warning that something in the sync pipeline is falling behind or silently failing.
- **Reconciliation jobs as a safety net.** Even a well-built event-driven sync pipeline should be paired with a periodic (daily or weekly) full reconciliation pass that catches drift the event stream missed — a dropped webhook, a permissions change that silently blocked a sync, or a source system outage during which changes queued up incorrectly.

---

## 33.6 A Note of Honesty — Perfect Freshness Isn't Achievable

It's worth being direct about the limits here, because "real-time knowledge" is sometimes promised more confidently than it can be delivered.

- **There is always some lag** between a source change and its reflection in the index — embedding, indexing, and propagation all take non-zero time, even in a well-optimized pipeline. "Real-time RAG" really means "low-lag RAG."
- **Change detection is never perfectly reliable.** Timestamps can be wrong or missing, webhooks can drop, and content hashing adds its own overhead. Some staleness will slip through any system, which is exactly why the reconciliation pass in Section 33.5 isn't optional.
- **Deletes are the easiest thing to get wrong**, precisely because they're the least visible failure — an orphaned vector doesn't throw an error, it just sits there being retrievable long after it should be gone. Test delete handling as deliberately as you test add and update handling.
- **Incremental indexing adds real system complexity** — change detection, partial-update logic, versioning metadata, and reconciliation jobs are all more moving parts than a simple nightly rebuild. For a small or slow-changing corpus, that complexity may not be worth it; know your actual freshness requirement before building for one you don't have.

---

## 33.7 Chapter Summary

- Full re-indexing on every change doesn't scale because it wastes cost on unchanged documents and creates unacceptable **freshness lag** for fast-moving corpora.
- **Incremental indexing** requires three capabilities: reliably detecting changed or deleted documents, updating only the affected chunks, and — critically — avoiding **orphaned vectors** left behind by deleted source content.
- Change detection commonly combines timestamps, content hashing, or CDC/webhooks with a periodic full-diff reconciliation pass as a safety net.
- **Versioning metadata** — document ID, version/hash, effective timestamp, and superseded status — must be tracked per chunk, extending the metadata discipline from Chapter 7 and enabling the conflict-resolution logic from Chapter 19 to actually function.
- Sync frequency should be **tiered by source volatility**, with bursty sources batched to protect cost and index stability rather than synced fully in real time.
- **Sync lag** deserves its own monitored metric, feeding directly into the drift detection practices from Chapter 32.
- Perfect real-time freshness isn't achievable — some lag, some missed change events, and real added system complexity are the honest cost of incremental indexing, and a slower rebuild cadence is a legitimate choice for corpora that don't need better.

**Coming up next (Chapter 34):** a knowledge base that's kept fresh and in sync is also a knowledge base that's exposed, request after request, to whatever content and access patterns flow through it — next we look at the security risks specific to RAG, from prompt injection hidden inside retrieved documents to access control that has to live inside the retrieval step itself.
