# Chapter 6: Chunking Strategies

## 6.1 What This Chapter Covers

Once a document has been cleanly parsed (Chapter 5), it still can't be handed to the retrieval system as one giant blob of text. It needs to be broken into smaller pieces — **chunks** — that can be individually embedded, indexed, and retrieved.

This chapter asks: how do you decide *where to cut*? It turns out this seemingly small decision has an outsized effect on retrieval quality — arguably more than the choice of embedding model itself. We'll walk through the major chunking strategies, what tradeoffs each one makes, and a newer idea called late chunking that rethinks the problem entirely.

---

## 6.2 Why Chunk Size and Boundaries Matter So Much

Think of chunking like cutting a long recipe book into index cards for a chef to flip through mid-cook. Cut too small, and a single card might say "Add 2 tablespoons and stir" — useless without knowing 2 tablespoons of *what*, or which dish it belongs to. Cut too large, and a single card might contain three entire unrelated recipes, forcing the chef to hunt through irrelevant text just to find the one relevant instruction.

RAG chunking faces exactly this tension:

- **Chunks that are too small** lose surrounding context. A sentence like "This increased by 12% year over year" is meaningless without knowing *what* increased. Retrieval might find the sentence but the generator has nothing to reason with.
- **Chunks that are too large** dilute relevance. A large chunk mixes relevant and irrelevant material together, which (a) makes the embedding vector a blurry average of multiple topics, hurting retrieval precision, and (b) wastes limited context window budget (a problem we'll dig into further in Chapter 18) with content the model didn't actually need.

> **Key idea:** A chunk should be large enough to be *self-contained and meaningful* on its own, and small enough to be *focused on a single idea* — chunking is fundamentally a search for that balance, and the right balance depends on your documents and your queries.

There is no universally correct chunk size. A legal contract, a customer support FAQ, and a codebase all have different natural "units of meaning," and good chunking strategies respect that rather than applying one fixed rule everywhere.

---

## 6.3 Fixed-Size Chunking

The simplest approach: pick a chunk size (say, 500 tokens) and cut the document into consecutive blocks of that size, optionally with some overlap between consecutive chunks so context isn't abruptly severed at a boundary.

- **How it decides where to cut:** Purely by character or token count, regardless of sentence or paragraph boundaries.
- **Strength:** Trivial to implement, fast, predictable chunk sizes (useful for cost and context-window budgeting).
- **Weakness:** Cuts can land mid-sentence or mid-idea, splitting a coherent thought across two chunks and hurting both chunks' embedding quality.

Fixed-size chunking is a reasonable baseline and is still widely used in practice — mainly because it's simple and fast — but it treats text as an undifferentiated stream of characters, which real documents rarely are.

---

## 6.4 Recursive Chunking

Recursive chunking tries to respect natural text boundaries while still targeting a roughly fixed size. The algorithm attempts to split on the "biggest" natural boundary first (e.g., paragraph breaks), and only falls back to smaller boundaries (sentences, then words, then raw characters) if a piece is still too large after that split.

- **How it decides where to cut:** A prioritized list of separators — paragraphs, then sentences, then words — applied recursively until pieces fit the target size.
- **Strength:** Produces chunks that respect sentence and paragraph structure far more often than pure fixed-size cutting, with only a little more implementation complexity.
- **Weakness:** Still fundamentally size-driven; it can still merge two unrelated paragraphs into one chunk just because they happen to fit the token budget together.

This is a common default in many RAG frameworks precisely because it's a good balance of simplicity and quality — it's a meaningfully better fixed-size chunker, not a fundamentally different idea.

---

## 6.5 Sliding Window Chunking

A sliding window is less a distinct cutting rule and more a modifier applied on top of fixed-size or recursive chunking: instead of chunk boundaries being back-to-back, consecutive chunks **overlap** by some amount (e.g., the last 50 tokens of chunk N are repeated as the first 50 tokens of chunk N+1).

- **How it decides where to cut:** Same as the underlying method (fixed-size or recursive), but with deliberate redundancy at the boundaries.
- **Strength:** Reduces the chance that a critical sentence gets awkwardly split with half its context in one chunk and half in another — a query matching content near a boundary is more likely to retrieve a chunk that has the full context.
- **Weakness:** Increases total storage and embedding cost (you're embedding some text twice), and if overlap is too large, it starts to look like the "too big, diluted" problem from a different angle — near-duplicate chunks cluttering retrieval results.

---

## 6.6 Structure-Aware Chunking

Rather than treating a document as a flat stream of text, structure-aware chunking uses the document's own **hierarchy** — headings, subheadings, sections, list boundaries — as the primary signal for where to cut. A markdown document with `#`, `##`, and `###` headers, for instance, naturally suggests where one topic ends and another begins.

- **How it decides where to cut:** Along structural boundaries already present in the document (headings, sections, list items, table boundaries), rather than a fixed size.
- **Strength:** Chunks tend to align with actual units of meaning the document's author intended — a chunk is "the introduction section" or "step 3 of the installation guide," not an arbitrary slice.
- **Weakness:** Requires the document to actually *have* reliable structure (which, as Chapter 5 discussed, isn't guaranteed even in "structured" formats like DOCX), and section lengths can vary wildly — some sections are one sentence, others are ten pages, which reintroduces the size-tension from section 6.2.

In practice, structure-aware chunking is often combined with recursive chunking as a second pass: split by structure first, then apply size-based recursive splitting *within* any section that's still too large.

---

## 6.7 Semantic Chunking

Semantic chunking tries to cut based on **meaning** rather than structure or size. A common approach: embed individual sentences (or small groups of sentences), then walk through the document measuring how much the meaning "drifts" from one sentence to the next. When the drift crosses a threshold — signaling a topic shift — that's where a chunk boundary is placed.

- **How it decides where to cut:** By detecting semantic discontinuity between adjacent pieces of text, using embedding similarity as the signal.
- **Strength:** Chunk boundaries can align with genuine topic shifts even in unstructured, free-flowing text (like a transcript or an essay) where there are no convenient headings to lean on.
- **Weakness:** Computationally more expensive (it requires embedding at a fine granularity just to *decide* how to chunk, before the "real" chunk embeddings are even created), and the drift threshold is a tunable hyperparameter that doesn't generalize perfectly across document types.

---

## 6.8 Comparing the Strategies

| Strategy | Cuts based on | Pros | Cons |
|---|---|---|---|
| **Fixed-size** | Character/token count | Simple, fast, predictable size | Ignores meaning; can split mid-idea |
| **Recursive** | Natural separators, size-bounded | Better boundary respect, still simple | Still fundamentally size-driven |
| **Sliding window** | Same as base method + overlap | Preserves context across boundaries | Redundant storage/embedding cost |
| **Structure-aware** | Document hierarchy (headings, sections) | Aligns with author's intended units | Requires reliable structure; uneven sizes |
| **Semantic** | Meaning/topic drift | Boundaries reflect real topic shifts | Expensive; threshold tuning needed |
| **Late chunking** | Full-document context, chunk boundaries applied after embedding | Chunk vectors retain document-level context | Newer technique, model/tooling support still maturing |

No single row in this table is "correct" — production systems frequently combine several: structure-aware splitting for the coarse boundaries, recursive splitting to bound size within a section, and sliding-window overlap at the edges.

---

## 6.9 Late Chunking — A Different Way to Think About the Problem

Every strategy above shares an assumption: **chunk first, embed second.** You decide the text boundaries, then generate an embedding for each resulting piece in isolation. This has a subtle cost — when a chunk is embedded on its own, it loses awareness of the document it came from. A chunk that says "The committee rejected this approach for the reasons outlined above" has no idea what "this approach" or "the reasons above" actually refer to, because that context lived in a different chunk.

**Late chunking** flips the order: **embed first, chunk second.**

1. Run the *entire* document through a long-context embedding model, generating token-level (or fine-grained) representations that are each influenced by the full document's context — because the model processed the whole document at once, every token's representation is colored by everything around it.
2. *Then* decide chunk boundaries — using any of the strategies above — and pool the relevant token-level representations within each chunk's span into a single chunk vector.

The result: each chunk's final embedding still reflects the meaning of the boundary decision, but was computed with full awareness of the surrounding document, not in isolation. Our earlier ambiguous chunk — "The committee rejected this approach for the reasons outlined above" — would, under late chunking, carry embedding signal informed by what "this approach" and "the reasons" actually were, even though the chunk's own text never restates them.

Late chunking is not a replacement for deciding *where* to cut — you still need one of the strategies from earlier in this chapter to define chunk boundaries. What it changes is *when* the embedding model gets to see document-level context relative to those boundaries. It's best understood as a complementary technique layered on top of a chunking strategy, not a competing seventh row that replaces the others.

---

## 6.10 Chunking Is Not a Solved Problem

It's tempting to treat chunking as a mechanical preprocessing step you configure once and forget. In practice:

- **There is no universal "best" chunk size.** The right size depends on document type, query patterns, and the embedding model in use — what works well for FAQ-style support articles often fails on dense legal contracts.
- **Chunking decisions are hard to evaluate in isolation.** A chunking strategy's real quality only shows up downstream, in retrieval and generation metrics (Chapters 27 and 28) — it's rarely obvious from looking at the chunks alone whether a boundary choice was good.
- **Structure-aware and semantic chunking assume clean input.** Both lean on either reliable document structure or coherent prose — and, as Chapter 5 covered, real-world documents don't always cooperate.
- **Late chunking depends on long-context embedding models.** It requires embedding models capable of processing full documents coherently (see Chapter 9), and tooling and best practices around it are still actively evolving.
- **More overlap and finer granularity aren't free.** Every strategy that improves context preservation tends to trade off against storage, compute cost, and index size at scale.

Chunking, like parsing, is one of those "unglamorous" pipeline stages that quietly determines whether the more sophisticated pieces of your system — re-ranking, agentic retrieval, evaluation — ever get a fair chance to work well.

---

## 6.11 Chapter Summary

- Chunking breaks parsed documents into retrievable units, and the choice of chunk size and boundaries has an outsized effect on retrieval quality.
- **Fixed-size chunking** splits by character/token count alone — simple but ignores meaning.
- **Recursive chunking** respects natural separators (paragraphs, sentences) while still targeting a size budget.
- **Sliding window** chunking adds overlap between consecutive chunks to avoid severing context at boundaries.
- **Structure-aware chunking** uses a document's own headings and sections as cut points, producing chunks aligned with author intent.
- **Semantic chunking** detects topic drift via embedding similarity to place boundaries at genuine meaning shifts.
- **Late chunking** embeds the full document first and derives chunk vectors afterward, preserving document-level context that isolated per-chunk embedding loses.
- No single strategy is universally best — production systems often combine structure-aware, recursive, and overlap techniques together.
- Chunking quality is best judged indirectly, through downstream retrieval and generation metrics, not by inspecting chunks alone.

**Coming up next (Chapter 7):** raw chunk text alone isn't enough to power precise, trustworthy retrieval — next we'll look at how attaching metadata to each chunk (entities, summaries, and document lineage) unlocks filtering, citation, and access control.
