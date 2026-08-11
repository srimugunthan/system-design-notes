# Chapter 8: Handling Multi-modal Content

## 8.1 What This Chapter Covers

So far, this Part has quietly assumed that documents are made of text — parse it (Chapter 5), cut it into chunks (Chapter 6), and tag it with metadata (Chapter 7). But real documents are rarely pure text. They contain charts, screenshots, diagrams, tables, and code blocks — content that doesn't flatten cleanly into a paragraph without losing something important.

This chapter asks: how do you fold these non-text elements into a *mostly text-based* RAG pipeline, without either discarding them or breaking the pipeline that was built around plain text? We'll treat this as a preprocessing problem — how to turn images, tables, and code into something a text-based retrieval system can meaningfully index and use. Chapter 24 will pick up the more advanced version of this problem: retrieving and reasoning over these content types *directly*, rather than converting them to text first.

---

## 8.2 Why Multi-modal Content Breaks the Text-Only Assumption

Picture handing our well-read student (Chapter 1) a page that's mostly a bar chart, with the caption "Figure 3: Regional growth trends." If you strip out the chart and hand the student only the caption, they know a chart existed — but not what it showed. If you skip the page entirely, the student never learns Figure 3 existed at all. Neither option preserves the actual information.

This is the core tension of multi-modal content in a text-based RAG pipeline: **the information is real and often important, but it isn't naturally expressed as prose.** A chart shows a trend visually. A table shows relationships positionally. A code block shows logic structurally. Forcing all three into flat paragraph text, without care, tends to either discard the information or mangle it into something no embedding model can usefully represent.

> **Key idea:** The goal of multi-modal preprocessing isn't to make every image and table *look like* text — it's to produce a text-based representation that preserves enough of the original meaning to be retrievable and useful, while acknowledging some fidelity loss is often unavoidable in a text-first pipeline.

---

## 8.3 Images: Captioning and Description Generation

For an image — a photo, diagram, screenshot, or chart — the most common preprocessing strategy in a text-based pipeline is to generate a **textual description** of it, using a vision-capable model, and treat that description as the retrievable content.

A few patterns are common in practice:

- **Simple captioning** — a short, one-sentence description ("A bar chart comparing quarterly revenue across four regions"). Cheap, but often too vague to be useful for retrieval — many different charts could share a caption this generic.
- **Detailed description generation** — a longer, structured description that attempts to capture the actual content of the image, not just its type: what the chart's axes represent, what the key trend or takeaway is, what text appears in a screenshot, what a diagram's components and relationships are. This is far more useful for retrieval, at the cost of a more expensive model call per image.
- **Contextual captioning** — incorporating the surrounding document text (the paragraph before and after the image, any caption or figure label) into the description-generation prompt, so the resulting description reflects not just what's visually present but what the image *means in context*. A chart captioned generically as "a line graph trending upward" becomes far more retrievable when the description folds in the surrounding sentence: "a line graph showing customer churn declining after the Q2 product change."

Whichever approach is used, the resulting description becomes a chunk like any other — embeddable, taggable with metadata (Chapter 7), and citable back to its source image. It's common practice to store a pointer to the original image alongside the generated description, so that a downstream system — or Chapter 24's more advanced multi-modal architectures — can retrieve the actual image later if needed, not just the text standing in for it.

---

## 8.4 Tables: Serialization vs. Structured Retrieval

Chapter 5 already introduced the core problem with tables: naive parsing flattens them into unreadable "text soup" that loses row/column relationships. Once a table has been parsed *correctly* — as structured rows and columns rather than flattened text — there are two broad strategies for making it retrievable.

**Table-to-text serialization** converts each row (or each cell) into a natural-language sentence that restates the row/column relationship explicitly, rather than relying on positional layout. For example, a row from a pricing table might become: *"For the Enterprise plan, the monthly price is $499 and the included seats are 50."* This produces something an embedding model can handle like any other chunk of prose, and it's simple to integrate into an existing text pipeline. The tradeoff is verbosity — serializing every row this way can balloon a compact table into a large amount of repetitive text, and cross-row comparisons ("which plan is cheapest?") remain hard for a model to do reliably from serialized prose alone.

**Structured table retrieval** keeps the table as an actual structured object — stored as, say, a small JSON or CSV representation alongside a lightweight text summary of what the table contains — and retrieves the *whole table* as a unit when relevant, rather than trying to make individual rows independently searchable. The generator then receives the table in structured form and can reason over it more reliably (especially if it's also capable of writing and executing code — a capability we'll touch on in Chapter 20's discussion of structured output and function calling). The tradeoff here is that retrieval has to correctly identify "this query needs table X as a whole," which is a coarser retrieval granularity than chunk-level text search is typically built for.

In practice, many pipelines use both: a text summary of the table (for retrieval matching) paired with the structured table data (for the generator to actually reason over once retrieved).

---

## 8.5 Charts and Figures

Charts deserve a brief note distinct from generic images, because they carry a specific kind of information: **quantitative relationships**, not just visual content. A bar chart isn't just "an image with bars in it" — it's a claim about specific numbers.

Where possible, the ideal preprocessing for a chart goes beyond a visual caption and attempts to extract the **underlying data**: the categories, the values, the trend. This might mean:

- Using a vision-capable model specifically prompted to read off approximate values from the chart, not just describe its shape
- Checking whether the source document (especially in formats like DOCX or HTML) embeds the chart's underlying data table alongside the rendered image — in which case, extracting that underlying data directly is far more reliable than trying to read it back out of pixels
- Falling back to a qualitative description ("shows a steady upward trend from 2023 to 2026") when precise value extraction isn't reliable, and being honest with downstream consumers — via metadata — that this is an approximate description rather than exact data

This is genuinely hard, and precision from chart-reading models varies a great deal depending on chart complexity and image quality — a caveat worth flagging explicitly to anyone consuming this pipeline's output, rather than presenting extracted chart values as ground truth.

---

## 8.6 Code Blocks: A Distinct Chunk Type

Code embedded in documentation, wikis, or technical reports — or entire source files in a codebase serving as a RAG knowledge base — needs fundamentally different handling than prose. The chunking strategies from Chapter 6 were designed around sentences and paragraphs; code doesn't have either.

A few reasons code needs its own treatment:

- **Character-count or sentence-based chunking breaks code semantics.** Cutting a function in half at an arbitrary character boundary produces two chunks, neither of which is valid, runnable, or independently meaningful — unlike prose, where a mid-paragraph cut at least leaves two readable (if incomplete) thoughts.
- **The natural unit of meaning is structural, not size-based.** A function, a class, or a module is the code equivalent of a paragraph — chunking by these structural units (using a language parser to find function and class boundaries, similar in spirit to the structure-aware chunking from section 6.6) preserves logical completeness far better than any fixed-size rule.
- **Code often needs its imports or class context to make sense.** A method chunk that references `self.config` is hard to interpret without knowing what `config` is — some pipelines attach a small amount of surrounding context (the enclosing class signature, relevant imports) as metadata or a prefix, rather than relying on the chunk being fully self-contained.
- **Comments and docstrings are valuable retrieval signal.** A well-written docstring often describes *intent* in a way raw code doesn't — treating the docstring as a natural-language summary of the code chunk (similar to the per-chunk summaries from Chapter 7) can meaningfully improve retrieval for natural-language queries like "how do we handle retry logic here?"

Chapter 37 will return to this topic in much more depth, covering RAG systems built specifically for code and technical documentation; this section's job is only to flag that code is not "text with different syntax" from a chunking perspective — it needs its own rules.

---

## 8.7 Cost and Latency at Ingestion Time

Every technique in this chapter shares a quiet cost: turning an image or table into a rich text representation usually means an extra model call at ingestion time, on top of the parsing work from Chapter 5. That cost is easy to underestimate when prototyping on a handful of documents and easy to feel painfully when ingesting tens of thousands of pages.

A few practical consequences worth planning for:

- **Vision-model captioning calls are typically the most expensive step in multi-modal ingestion**, per unit of content — often more expensive than embedding the resulting text. A document with fifty screenshots can cost far more to ingest than a fifty-page pure-text document of similar length.
- **Not every image is worth this cost.** A logo repeated on every page of a document, or a purely decorative icon, adds ingestion cost without adding retrievable information. Simple heuristics — filtering by image size, position on the page, or duplicate-image detection — can meaningfully cut cost without losing real content.
- **Batching and caching help.** If the same source document is re-ingested after a minor text edit elsewhere, there's no need to re-caption images that haven't changed — content-hashing images and caching their descriptions avoids redundant model calls (a specific case of the incremental indexing problem Chapter 33 covers more generally).
- **Table serialization and code chunking are comparatively cheap** — they're mostly deterministic transformations rather than model calls — which is one more reason to prefer structural, rule-based handling over reaching for a vision or language model call whenever a simpler transformation will do.

The broader lesson: multi-modal preprocessing quality and multi-modal preprocessing cost trade off directly against each other, and that tradeoff deserves the same deliberate attention given to chunk size (Chapter 6) or embedding model choice (Chapter 9) — it's not something to leave as a default setting.

---

## 8.8 What This Chapter Does Not Solve

It's worth being explicit about the boundary of what's covered here versus what's ahead. Everything in this chapter converts multi-modal content *into text* so it can flow through a conventional, text-based RAG pipeline. This is a pragmatic, widely used approach — but it has real limits:

- **Conversion is lossy by nature.** A generated image description, however detailed, is not the image. A user asking a highly specific visual question ("is the line in the chart dashed or solid?") may not get a reliable answer if all the system has is a paraphrased description.
- **The generator never actually "sees" the original content** under this approach — it reasons only over the text stand-in, inheriting any gaps or errors introduced during description generation or serialization.
- **This is fundamentally different from retrieving and reasoning over images, tables, and charts directly** using models capable of processing them natively at query time — an architecture Chapter 24 covers under the name of full multi-modal RAG, where retrieval can return an actual image to a vision-capable generator instead of a text description of one.

Think of this chapter's techniques as the pragmatic, lower-cost on-ramp: they let a text-based pipeline handle multi-modal source material reasonably well, without requiring a fully multi-modal retrieval and generation architecture. For many use cases, that's sufficient. For others — heavy visual or tabular reasoning — the more advanced architecture in Chapter 24 becomes necessary.

---

## 8.9 Chapter Summary

- Multi-modal content — images, tables, charts, and code — doesn't flatten cleanly into prose without risking loss of the information it's meant to convey.
- **Images** are typically handled via generated captions or descriptions, ideally informed by surrounding document context, with a pointer kept back to the original image.
- **Tables** can be handled via **serialization** (turning rows into explicit natural-language sentences) or **structured retrieval** (keeping the table as a retrievable structured object) — many pipelines use both together.
- **Charts** carry quantitative meaning, not just visual content, so preprocessing should attempt to extract underlying data or values where possible, and flag approximate extractions honestly via metadata.
- **Code requires its own chunking rules** — structural units like functions and classes, not character counts — because arbitrary cuts break code's logical completeness in a way prose can tolerate.
- Docstrings and comments are valuable retrieval signal for code chunks, functioning much like the per-chunk summaries introduced in Chapter 7.
- All techniques in this chapter convert multi-modal content **into text**, which is pragmatic but inherently lossy — the generator reasons over a text stand-in, not the original content.
- **Chapter 24** covers the more advanced alternative: retrieving and reasoning over images, tables, and charts directly, without first converting them to text.

**Coming up next (Chapter 9):** with parsed, chunked, metadata-enriched, and multi-modal-aware content ready, Part III turns to how this content actually gets represented numerically — starting with how embedding models work and how to choose one for your RAG system.
