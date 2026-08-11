# Chapter 24: Multi-modal RAG

## 24.1 What This Chapter Covers

Chapter 8 introduced the groundwork for handling multi-modal content — extracting text from images, describing charts, transcribing audio — so that everything ultimately became text a standard RAG pipeline could index and retrieve. That's a reasonable and often sufficient approach. But converting everything to text loses something: a chart's precise shape, a diagram's spatial layout, a product photo's actual visual detail.

This chapter goes further and answers the question: **what does it look like to retrieve and reason over images, tables, and video directly — not just their text descriptions — and what does that cost you in return?**

---

## 24.2 Why "Convert Everything to Text" Isn't Always Enough

The Chapter 8 approach — caption the image, describe the chart, transcribe the audio, then treat the result as ordinary text — works well when the content's meaning survives translation into words. A scanned invoice's line items survive OCR just fine. A slide with a bulleted list survives captioning fine.

It works much less well when the *visual* information itself is the answer. Consider a user asking "which quarter shows the steepest revenue decline in this chart?" A caption like "a line chart showing quarterly revenue from 2023 to 2025" doesn't contain the answer — the answer requires actually looking at the shape of the line. Or consider "does this product photo show scratches on the casing?" — no caption pipeline reliably captures every visual detail a downstream question might ask about, because the person writing the caption didn't know which detail would later matter.

> **Core idea:** converting images to text captions is a lossy compression step. It's the right tradeoff for many use cases, but for questions where the visual detail itself is the answer, you need to retrieve and reason over the actual image — not a paraphrase of it.

Multi-modal RAG is the set of techniques for doing exactly that: keeping images, tables, and video as first-class retrievable objects rather than flattening everything to text upfront.

---

## 24.3 Multi-modal Embeddings: A Shared Space for Text and Images

The technical foundation that makes this possible is the **multi-modal embedding model** — a model trained to place text and images into the *same* vector space, such that an image and a text description of that image end up with similar embeddings, even though one started as pixels and the other as words.

This is a meaningfully different capability from the text embedding models in Chapter 9. A text embedding model maps text to vectors; a multi-modal embedding model (CLIP-style architectures are the best-known family) maps both text *and* images into one shared space, trained so that matching image/caption pairs land close together and mismatched pairs land far apart.

The practical payoff: you can embed a user's *text* query ("a red sports car parked in front of a brick building") and run similarity search directly against a store of *image* embeddings — no captioning step required, no lossy text intermediary. The image itself is the retrievable unit, indexed the same way a vector database indexes text chunks (Chapter 10), just embedded by a different model.

This also opens up **image-to-image** and **image-to-text** retrieval, not just text-to-image: given a product photo, find similar products; given a diagram, find the section of a manual that discusses it. The retrieval direction becomes flexible because everything shares one geometric space.

---

## 24.4 Passing Raw Images to the Generator

Embedding is only half the story — retrieval finds the right image, but generation needs to actually reason over it. This is where **multi-modal LLMs** (models that accept images as direct input alongside text, not just text descriptions of images) come in.

The augmentation step from Chapter 1's four-box pipeline changes shape: instead of "combine the question and retrieved text chunks into one prompt," it becomes "combine the question, retrieved text chunks, *and* retrieved images into one multi-modal prompt." The generator then reasons over the actual pixels — reading values off a retrieved chart, comparing two retrieved product photos, reading a table's structure directly rather than a linearized text version of it — the same way a person would look at the retrieved evidence rather than read a description of it.

Tables deserve a specific mention here, since they sit awkwardly between text and image. A table converted to text (Markdown or CSV-style) usually preserves its data faithfully and is often the *better* choice — it's cheaper to embed, cheaper to pass to the generator, and text-based table representations are frequently sufficient for questions like "what was Q3 revenue?" But a table's original visual layout can carry information a flattened text version loses — merged cells, visual grouping, footnote markers, a nested header structure. For most factual lookups, text extraction remains the right default; reserve retrieving the table as an image for cases where visual layout materially aids the answer, such as understanding a nested header spanning multiple sub-columns.

---

## 24.5 Video: A Compound Modality Problem

Video RAG isn't really "one more modality" — it's several modalities stacked on top of each other, each with its own retrieval challenge:

- **Transcript** — the spoken-word content, typically extracted via speech-to-text, and retrievable as ordinary text chunks the same way you'd chunk a document (Chapter 6)
- **Visual frames** — what's actually shown on screen, which the transcript often doesn't capture at all (a demo video showing a UI walkthrough might have almost no narration during the key moment)
- **Timestamps** — the critical piece of metadata that ties a retrieved moment back to a specific point in the source video

The practical approach most systems take is a hybrid: chunk the transcript for text-searchable content, periodically sample frames (e.g., every few seconds, or at scene changes) and embed them with a multi-modal embedding model for visually-searchable content, and tag every chunk — transcript segment or sampled frame — with its **timestamp range**.

That timestamp is what makes citation meaningful in video RAG. A citation to "paragraph 3 of document.pdf" is straightforward; a citation to "this video" without a timestamp is nearly useless to a user who now has to scrub through a 40-minute recording to find the relevant 15 seconds. A good video RAG system returns not just "yes, this video covers that topic" but "at 12:34, the presenter shows exactly this" — often with the retrieved frame itself displayed alongside the timestamp so the user can verify at a glance before jumping to that point.

Frame sampling itself is a real design tradeoff: sample too sparsely and you miss the one frame that actually shows the answer (a chart that's on screen for two seconds); sample too densely and indexing cost and storage balloon, since embedding every frame of every video in a large library is expensive at scale.

---

## 24.6 Adapting Retrieval Strategies for Multi-modal Content

The retrieval strategies covered in Part IV weren't designed with images in mind, but most of them carry over with some adaptation rather than needing to be reinvented from scratch.

**Hybrid text-and-image retrieval** is the most common pattern: run text search (Chapter 12) against transcripts, captions, and surrounding document text, run multi-modal similarity search against image embeddings, and fuse the two ranked lists — the same reciprocal rank fusion idea from Chapter 12 applies here just as well to combining a text-based ranking with an image-based one, since RRF doesn't care what produced the underlying scores.

**Re-ranking (Chapter 15) gets a multi-modal upgrade too.** A vision-capable cross-encoder can look at the query and a candidate image *together* — the same principle as a text cross-encoder, just operating on pixels instead of tokens — to judge relevance more precisely than the initial embedding-based retrieval pass. This matters more here than in text RAG, because multi-modal embedding similarity (Section 24.3) is comparatively noisier, so a strong second-pass reranker earns its cost more readily.

**MMR-style diversity (Chapter 14) applies directly to image results.** Returning five near-duplicate product photos is exactly as unhelpful as returning five near-duplicate text chunks — a diversity-aware selection step is just as relevant for a gallery of retrieved images as it is for a list of retrieved passages.

One genuinely new wrinkle: **query modality doesn't have to match result modality.** A user's query is almost always text, but the most useful retrieved evidence might be an image, a table, or both. Deciding *when* to route a query toward image retrieval versus text retrieval — or both, in parallel — is itself a small query-understanding problem, related to but distinct from the query rewriting and decomposition techniques in Chapter 13.

---

## 24.7 The Honest State of Multi-modal RAG Today

It's worth being direct about where this technology currently stands relative to text RAG, because the gap is real and affects how you should scope a project.

**Retrieval quality is less mature.** Text embedding models have years more refinement, benchmarking, and production hardening behind them than multi-modal embedding models. Multi-modal similarity search is more prone to surprising failures — retrieving an image that shares color or composition with the query but not actual semantic content, for instance.

**Cost per query is meaningfully higher.** Multi-modal LLM calls that accept images as input generally cost more in tokens (images consume a nontrivial token budget of their own) and latency than pure-text calls. A RAG pipeline that retrieves and passes three images alongside text context is paying a noticeably higher per-query bill than a text-only equivalent — this is a real budget-line item, not a rounding error, and worth explicitly modeling in the cost projections Chapter 31 will cover.

**Evaluation is harder.** Chapter 27 and 28 will discuss retrieval and generation metrics built primarily around text — measuring whether a retrieved chunk is relevant, whether a generated answer is faithful to its source. Applying the same rigor to "was the retrieved image actually relevant?" or "did the model correctly read the chart?" is a less standardized, less tooled-up problem, and often ends up requiring more human review (Chapter 29) than an equivalent text pipeline would.

**Storage and infrastructure are heavier.** Storing and serving raw images and video frames at scale — rather than just their text embeddings — is a meaningfully different infrastructure problem from text RAG, with real implications for storage cost and retrieval latency.

None of this is a reason to avoid multi-modal RAG when the use case genuinely needs it — a system answering questions about product photos, engineering diagrams, or video training material has no real substitute. It's a reason to scope multi-modal capability deliberately, to the specific content types and questions that need it, rather than defaulting every image and table in a corpus into a multi-modal pipeline when a text description would have served the question just as well at a fraction of the cost.

---

## 24.8 Chapter Summary

- Converting images and tables to text descriptions (Chapter 8's approach) is often sufficient, but it's lossy — questions where the visual detail itself is the answer need direct image retrieval and reasoning, not a paraphrase.
- **Multi-modal embedding models** place text and images in a shared vector space, enabling text-to-image, image-to-image, and image-to-text search without a captioning intermediary.
- **Multi-modal LLMs** accept retrieved images directly as generation input, reasoning over actual pixels rather than a text description of them.
- Tables sit between text and image — text extraction is usually the better default for factual lookups; retrieving the table as an image is worth it mainly when visual layout (merged cells, nested headers) carries meaning that flattened text loses.
- Video RAG combines several modalities — **transcripts** (text-chunked), **sampled visual frames** (multi-modal embedded), and **timestamps** — with timestamps being essential for making a retrieved video moment actually useful and citable.
- Multi-modal RAG today is **less mature, more expensive per query, and harder to evaluate** than text RAG — real infrastructure, cost, and quality tradeoffs that mean it should be scoped deliberately, not applied by default everywhere image or table content exists.

**Coming up next (Chapter 25):** we'll close out Part VI by looking at a different kind of retrieval target entirely — not an external knowledge base, but a growing store of what the system has learned about a user across past conversations.
