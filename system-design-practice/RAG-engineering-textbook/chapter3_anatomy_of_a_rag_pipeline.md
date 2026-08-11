# Chapter 3: Anatomy of a RAG Pipeline

## 3.1 What This Chapter Covers

Chapter 1 gave us a simple four-box diagram: retriever, augmentation, generator, answer. That picture is correct as far as it goes, but it hides a lot of real engineering work behind the word "retriever" — work that determines whether a RAG system actually performs well or quietly fails.

This chapter opens that box up. By the end, you'll have a five-stage mental model of a real RAG pipeline, know what each stage is responsible for, what commonly goes wrong at each one, and where in this book we tackle each problem in depth.

---

## 3.2 From Four Boxes to Five Stages

Chapter 1's diagram compressed everything that happens *before* a question is even asked into a single word: "knowledge base." That's fine for a first mental model, but in a real system, getting documents into a searchable, useful state is most of the engineering effort — often more than the retrieval and generation steps combined.

If we split "retriever searches a knowledge base" into its two honest halves — *building* the knowledge base, and *searching* it — we get five stages instead of four:

1. **Ingestion** — get raw documents into the system and parse them into usable text
2. **Chunking & Indexing** — break text into pieces and store them so they can be searched efficiently
3. **Retrieval** — given a question, find the most relevant pieces
4. **Augmentation** — assemble the question and retrieved pieces into a prompt
5. **Generation** — the LLM produces the final answer

Stages 1 and 2 happen *before* any user ever asks a question — often on a schedule, or whenever new documents arrive. Stages 3 through 5 happen *at query time*, in response to a specific question. Keeping this offline/online split in mind will save you a lot of confusion later, because the two halves are debugged very differently: one is a data pipeline, the other is closer to a live web request.

---

## 3.3 Stage 1: Ingestion & Parsing

Every RAG system starts with raw source material — PDFs, web pages, Word documents, Slack exports, database rows, scanned images, transcripts. **Ingestion** is the process of pulling this material in and converting it into clean, structured text the rest of the pipeline can work with.

This sounds mundane. It is not. A PDF with a two-column layout, embedded tables, and a footer on every page is a genuinely hard parsing problem — get the column order wrong and you'll silently scramble sentences that were never adjacent in the original document.

**What can go wrong here:**

- Multi-column layouts get read left-to-right across columns instead of down each column, garbling text
- Tables get flattened into unreadable strings of numbers with no structure
- Headers, footers, and page numbers get mixed into the body text
- Scanned documents need OCR, which introduces its own error rate
- Images, charts, and diagrams get dropped entirely, even when they carry the actual answer

Get ingestion wrong, and every downstream stage inherits corrupted input — no amount of clever retrieval or prompting can recover information that was mangled or lost at this step. **Part II** (Chapters 5–8) is dedicated to this stage: document parsing, chunking, metadata extraction, and multi-modal content.

---

## 3.4 Stage 2: Chunking & Indexing

Once you have clean text, you can't hand an entire 300-page policy manual to the model on every query — it wouldn't fit in the context window, and even if it did, Chapter 2 already warned us that dumping huge amounts of text into a prompt is expensive and doesn't guarantee the model uses it well.

So documents get broken into smaller pieces, called **chunks** — typically a paragraph to a few paragraphs long. Each chunk is then converted into a numerical representation (an **embedding**, covered in Chapter 9) that captures its meaning, and stored in a structure optimized for fast similarity search (a **vector database** or **index**, covered in Chapters 10–11).

**What can go wrong here:**

- **Chunks that are too small** lose context — a sentence like "it must be filed within 30 days" is meaningless without knowing what "it" refers to
- **Chunks that are too large** dilute relevance — a chunk containing five unrelated topics makes it harder for retrieval to find the one sentence that actually answers the question
- **Chunk boundaries that split related information apart** — cutting a table row in half, or separating a claim from the footnote that qualifies it
- **Missing metadata** — without knowing a chunk's source, date, or section, you can't filter, cite, or reason about *which* version of a policy a chunk came from

This is arguably the highest-leverage stage in the entire pipeline: a well-chunked, well-indexed knowledge base makes retrieval's job easy, while a poorly chunked one makes even a great retrieval algorithm perform badly. **Part II and Part III** (Chapters 5–12) cover this in depth, including hybrid approaches that combine semantic and keyword search.

---

## 3.5 Stage 3: Retrieval

This is the stage most people picture when they hear "RAG." At query time, the user's question is compared against the index built in Stage 2, and the system returns the chunks judged most relevant.

But "compare the question against the index" hides real subtlety. A user's raw question is often not the ideal search query — it might be vague, contain typos, use different vocabulary than the source documents, or actually require *multiple* searches to answer fully (a question like "how did our refund policy change between 2023 and 2024" implicitly needs two different time periods retrieved).

**What can go wrong here:**

- The question and the relevant documents use different words for the same concept ("cancel my plan" vs. a policy document that says "terminate subscription"), and pure keyword search misses the match
- Semantic search finds text that's *topically* similar but doesn't actually answer the question
- Too few chunks are retrieved (missing part of the answer) or too many are retrieved (burying the relevant chunk in noise)
- Multi-part questions only get partially addressed by a single retrieval pass

**Part IV** (Chapters 13–16) covers this stage in full: understanding and rewriting queries, retrieval strategies, re-ranking retrieved results for relevance, and multi-hop retrieval for questions that need several rounds of searching.

---

## 3.6 Stage 4: Augmentation

**Augmentation** is the often-underestimated step of actually assembling the final prompt: combining the system instructions, the user's question, and the retrieved chunks into text the model will read.

This is more than string concatenation. Decisions made here directly shape the answer's quality:

- **How are chunks ordered?** Given what Chapter 2 flagged about "lost in the middle," where you place the most important chunk in the prompt can matter.
- **How much of the context budget goes to retrieved chunks vs. conversation history vs. instructions?**
- **How are chunks labeled?** Including source, date, or document title alongside each chunk lets the model — and the end user — know where an answer came from.
- **What instructions are given about how to use the context?** Should the model refuse to answer if the retrieved chunks don't contain the answer, or is it allowed to fall back on its own knowledge?

**What can go wrong here:** even with perfect retrieval, a poorly constructed prompt can bury the right chunk, fail to tell the model it's allowed to say "I don't know," or exceed the context window and get silently truncated. **Part V** (Chapters 17–20) covers prompt engineering for RAG, context window management, and handling messy or contradictory retrieved context.

---

## 3.7 Stage 5: Generation

Finally, the assembled prompt goes to the LLM, which produces the answer. This is the stage Chapter 1 focused on when introducing hallucination and grounding — the model reads the retrieved context and, ideally, bases its answer on it rather than on memorized (and possibly outdated) parametric knowledge.

**What can go wrong here:**

- The model ignores the retrieved context and answers from memory anyway
- The model blends retrieved facts with memorized facts, producing an answer that's *partly* right in a way that's hard to catch
- The model over-trusts a retrieved chunk that was itself wrong, outdated, or irrelevant
- The output format doesn't match what the downstream application expects (this becomes especially important when RAG feeds structured outputs or function calls, covered in Chapter 20)

Generation quality is also where many of the advanced architectures in **Part VI** (Chapters 21–25) come in — agentic RAG, self-correcting retrieval loops, and knowledge-graph-based approaches all exist to make this final step more reliable.

---

## 3.8 The Full Pipeline, End to End

```
                         OFFLINE (build the knowledge base)
                         ─────────────────────────────────
Raw Documents
     │
     ▼
[1] Ingestion & Parsing   ──►  clean, structured text
     │
     ▼
[2] Chunking & Indexing   ──►  chunks + embeddings stored in a searchable index
                         ─────────────────────────────────
                         ONLINE (answer a real question)
                         ─────────────────────────────────
User Question
     │
     ▼
[3] Retrieval             ──►  finds the most relevant chunks from the index
     │
     ▼
[4] Augmentation           ──►  assembles question + chunks + instructions into a prompt
     │
     ▼
[5] Generation (LLM)       ──►  reads the prompt and writes an answer
     │
     ▼
Final Answer (ideally with citations back to source chunks)
```

---

## 3.9 A Note of Honesty: The Pipeline Isn't Really a Straight Line

The diagram above is drawn as a clean, one-directional pipe, and for a first system, that's a reasonable way to build it. But real production RAG systems rarely stay that linear:

- Retrieval sometimes needs to run more than once per question (multi-hop retrieval, Chapter 16)
- Some systems check whether retrieved chunks are actually good enough *before* generating, and re-retrieve if not (Corrective RAG, Chapter 23)
- Some systems let the model decide, mid-answer, that it needs to search again — closer to an agent taking actions than a fixed pipeline (Agentic RAG, Chapter 21)
- Failures often aren't isolated to one stage — a bad chunking decision made months ago can quietly cause a retrieval failure today, and tracing the root cause back through the pipeline is a real production skill (Chapter 32)

Treat the five-stage diagram as your starting mental model, not a rigid architecture. Almost every advanced technique later in this book is best understood as "which stage of this pipeline are we improving, and why."

---

## 3.10 Chapter Summary

- Chapter 1's four-box diagram compresses two very different phases into "the retriever" — building the knowledge base and searching it deserve to be treated as separate stages.
- A realistic RAG pipeline has **five stages**: ingestion, chunking & indexing, retrieval, augmentation, and generation.
- **Ingestion and chunking & indexing happen offline**, ahead of any user question; **retrieval, augmentation, and generation happen online**, in response to a specific query.
- Each stage has its own characteristic failure modes — from garbled PDF parsing, to chunk boundaries that sever context, to retrieval missing relevant text, to prompts that bury the right chunk, to a model that ignores good context anyway.
- **Chunking and indexing quality is one of the highest-leverage points in the whole pipeline** — a great retrieval algorithm can't fully compensate for a poorly built index.
- Real production systems often make this pipeline **non-linear**, adding re-retrieval loops, quality checks, and agentic decision-making on top of the basic five-stage flow.

**Coming up next (Chapter 4):** before we go deeper into any single pipeline stage, we need a shared, practical understanding of how LLMs actually process text — tokens, context windows, and prompt structure — so that later chapters on chunking, context management, and generation all build on the same foundation.
