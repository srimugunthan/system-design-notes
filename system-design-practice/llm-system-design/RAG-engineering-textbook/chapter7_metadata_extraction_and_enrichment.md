# Chapter 7: Metadata Extraction & Enrichment

## 7.1 What This Chapter Covers

Imagine a librarian (recall the analogy from Chapter 1) who hands you the right page from the right book — but no cover, no title, no author, no publication date, and no indication of which library it even came from. You'd have the content, but you'd have no way to judge whether to trust it, cite it, or even confirm it's still relevant.

This chapter covers **metadata** — the structured information attached to each chunk *alongside* its text — and why a RAG system built on raw text chunks alone quickly runs into real limitations around filtering, trust, and access control.

---

## 7.2 Why Text Alone Isn't Enough

A chunk of text, by itself, answers the question "what does this say?" It does not answer:

- *Where did this come from?* (Which document, section, and source system?)
- *When was it written or last updated?* (Is this policy still current?)
- *Who is allowed to see this?* (Should this chunk even be eligible for retrieval for this particular user?)
- *What kind of content is this?* (A legal clause? A support FAQ answer? A code snippet?)
- *What entities does it discuss?* (Which company, product, person, or date is this actually about?)

Without answers to these questions, a retrieval system is forced to treat every chunk as an anonymous, interchangeable blob of text — searchable only by semantic similarity, with no way to narrow, verify, or restrict results. **Metadata** is the fix: structured fields attached to each chunk that capture exactly this contextual information.

> **Key idea:** Retrieval quality isn't only about finding text that *sounds* relevant — it's about finding text that's relevant, current, trustworthy, and permitted for this particular user, in this particular context. Metadata is what makes those last three qualities possible.

---

## 7.3 Where Metadata Comes From

Metadata generally falls into two categories, based on how it's obtained.

**Extracted metadata** comes largely for free from the source document and ingestion process:
- File-level facts: filename, source system, file type, ingestion timestamp
- Document-level facts pulled from the parser (Chapter 5): title, author, creation/modification date
- Structural facts from chunking (Chapter 6): which section or heading a chunk fell under, its position in the document

**Enriched metadata** is *generated* — typically by running an additional model pass over each chunk after it's created:
- **Entity tags** — people, organizations, products, dates, locations mentioned in the chunk
- **Per-chunk summaries** — a one- or two-sentence gloss of what the chunk says
- **Topic or category labels** — a classification of what kind of content this is (e.g., "pricing," "troubleshooting," "legal clause")

The distinction matters because extracted metadata is essentially free — it's already sitting in the document or the pipeline — while enrichment costs additional compute (usually an LLM call per chunk) and needs to be weighed against that cost at scale.

---

## 7.4 Entity Tagging

Entity tagging identifies the specific "things" a chunk is about — company names, product names, people, dates, monetary amounts, locations — and attaches them as structured fields rather than leaving them buried in prose.

Why this matters for retrieval: a purely semantic search for "the Q3 revenue figure" might return several chunks that all discuss revenue in general terms, without reliably distinguishing which one is actually about Q3. If each chunk carries an extracted entity like `quarter: Q3` or `fiscal_period: 2026-Q3`, the retrieval system can filter or boost using that structured signal directly, instead of hoping the embedding model captured the distinction precisely enough on its own.

Entity tagging also enables a second, often underrated use case: **cross-referencing**. If you know which chunks mention a specific product name, you can build features like "show me everything in the knowledge base that discusses Product X," independent of any specific user question — a capability that pure semantic search doesn't naturally provide.

---

## 7.5 Per-Chunk Summaries

A per-chunk summary is a short, generated description of what a chunk contains — not the retrieval content itself, but a compact gloss of it.

This sounds redundant (why summarize something you're about to retrieve anyway?) but it serves a few distinct purposes:

- **Improved retrieval matching.** Some retrieval setups embed the summary *instead of*, or *alongside*, the raw chunk text — a clean, information-dense summary can sometimes match a user's query more reliably than a noisy raw excerpt, especially when the raw chunk contains boilerplate or awkward parsing artifacts left over from Chapter 5.
- **Faster human review.** When a human is auditing retrieved chunks (see Chapter 29's discussion of human-in-the-loop evaluation), scanning short summaries is far faster than reading full chunk text.
- **Building indexes over indexes.** In hierarchical retrieval setups, chunk summaries can themselves be grouped and summarized again, forming a coarse-to-fine retrieval structure — start with document-level summaries, narrow to section-level, then finally to the chunk itself.

The cost, again, is real: generating a summary per chunk means an additional model call per chunk at ingestion time, which adds up at scale and needs to be budgeted for (see Chapter 31's treatment of cost optimization).

---

## 7.6 Hierarchical Metadata — Lineage From Chunk to Source

A single chunk rarely exists in isolation — it belongs to a section, which belongs to a document, which came from a source system. **Hierarchical metadata** captures this lineage explicitly, so that every chunk carries a trail back to its origin.

A typical lineage might look like:

```
chunk_id: doc_4471_chunk_12
  └── section: "3.2 Termination Clauses"
        └── document: "Master Services Agreement — Acme Corp"
              └── source: "Legal Contracts Repository"
                    └── ingested_at: 2026-03-14
```

This lineage matters for several concrete reasons:

- **Citation.** When the generator produces an answer, being able to say "according to Section 3.2 of the Acme Corp Master Services Agreement" is far more trustworthy — and far more verifiable — than a bare, unattributed chunk of text.
- **Debugging retrieval failures.** If a retrieved chunk seems irrelevant or wrong, lineage lets you trace exactly which document and section it came from, which is often the fastest way to diagnose whether the problem is in parsing, chunking, or retrieval itself.
- **Freshness tracking.** If a source document gets updated, lineage tells you exactly which chunks need to be re-processed — a capability Chapter 33 will build on when we discuss incremental indexing.

---

## 7.7 Metadata as a Retrieval Filter

Perhaps the most immediately practical use of metadata is **filtering** — narrowing the candidate pool before or after the semantic search step, rather than relying on embedding similarity alone to do all the work.

| Filtering approach | How it works | Example |
|---|---|---|
| **Pre-filter (before vector search)** | Restrict the searchable set using metadata, then run vector search only within that subset | Only search chunks where `department = "finance"` before ranking by similarity |
| **Post-filter (after vector search)** | Run vector search broadly, then discard results that fail a metadata condition | Retrieve top 50 candidates by similarity, then drop any where `document_date < 2024` |
| **Boosting** | Use metadata to adjust ranking scores rather than hard-excluding results | Slightly boost chunks tagged `source_type = "official_policy"` over `source_type = "forum_post"` |

Common real-world filters include date ranges ("only policies updated this year"), source type ("only official documentation, not community posts"), department or business unit, and language. Without metadata, none of these filters are possible — the system can only ask "what sounds similar to this query," never "what sounds similar to this query *and* meets these constraints."

---

## 7.8 Metadata and Access Control

Metadata also plays a quieter but critical role in **security**: not every chunk should be visible to every user. A chunk from an HR salary-band document, a legal case under privilege, or an internal incident report may need to be restricted to specific roles or teams.

By tagging chunks with access-control metadata — such as `allowed_roles`, `sensitivity_level`, or `department_owner` — a retrieval system can filter out chunks a given user isn't authorized to see, *before* those chunks are ever handed to the generator. This is a foundational building block for the broader security and guardrails discussion in Chapter 34, which covers this topic in much more depth, including the harder edge cases (what happens when a retrieved chunk is authorized, but the *answer* synthesized from it leaks something that shouldn't be shared).

For now, the key point is structural: access control in RAG is only possible if the necessary permission metadata exists on each chunk in the first place. It cannot be bolted on as an afterthought once chunks are already indexed without that information.

---

## 7.9 The Limits of Metadata Enrichment

Metadata is powerful, but it's not free, and it's not infallible. A few honest caveats worth internalizing:

- **Enrichment is not perfectly accurate.** Entity extraction and summarization are themselves model outputs — they can mislabel, miss entities, or generate summaries that subtly misrepresent the chunk. Treat enriched metadata as a strong signal, not ground truth.
- **Enrichment costs compound at scale.** Running an entity tagger and a summarizer over every chunk of a large corpus is a real, ongoing compute cost — one that needs to be weighed against the retrieval quality gain it actually delivers.
- **Stale metadata is a real risk.** If a source document is updated but the metadata pipeline isn't re-run, chunks can carry outdated tags — a wrong `department` or an outdated `sensitivity_level` is arguably worse than having no metadata at all, since it creates false confidence.
- **Over-filtering can hurt recall.** Aggressive metadata filters can accidentally exclude a genuinely relevant chunk because of a slightly wrong or missing tag — filtering is a precision/recall tradeoff like any other, not a free lunch.
- **Metadata schemas need real design discipline.** An unplanned, ad hoc metadata schema that grows organically tends to become inconsistent across document types, making filters unreliable exactly when they're needed most.

Metadata should be designed deliberately, validated periodically, and treated as an evolving part of the system — not a one-time enrichment script you run and forget.

---

## 7.10 Chapter Summary

- Raw chunk text alone cannot answer questions about **provenance, freshness, permissions, or content type** — this is the job of metadata.
- Metadata splits into **extracted metadata** (largely free, pulled from the document and pipeline) and **enriched metadata** (generated via additional model passes, at additional cost).
- **Entity tagging** attaches structured facts (people, products, dates) to chunks, enabling precise filtering and cross-referencing beyond what semantic similarity alone provides.
- **Per-chunk summaries** support cleaner retrieval matching, faster human review, and hierarchical coarse-to-fine retrieval structures.
- **Hierarchical metadata** (chunk → section → document → source) enables citation, retrieval debugging, and freshness tracking.
- Metadata enables **pre-filtering, post-filtering, and ranking boosts** — retrieval constraints that pure vector similarity cannot express on its own.
- **Access-control metadata** is foundational to RAG security, restricting which chunks are even eligible for retrieval by a given user — a topic Chapter 34 explores in depth.
- Enriched metadata is a **model output, not ground truth** — it carries its own error rate, cost, and staleness risks that must be actively managed.

**Coming up next (Chapter 8):** metadata helps us describe and filter text-based chunks, but not every piece of content is text in the first place — next we'll look at how to handle images, tables, charts, and code as part of a mostly text-based ingestion pipeline.
