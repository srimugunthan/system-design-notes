# Chapter 22: GraphRAG & Knowledge Graphs

## 22.1 What This Chapter Covers

Ask a standard RAG system "what is the refund policy?" and chunk-based vector retrieval does fine — the answer lives in one or two well-defined passages. Now ask it "how are Company A, Company B, and Company C connected?" and something breaks down, even if every fact needed to answer is somewhere in the knowledge base.

This chapter answers the question: **why does pure chunk-based vector retrieval struggle with relationship questions, and how does representing knowledge as a graph fix that?**

---

## 22.2 The Limits of Similarity Search for Relationships

Every retrieval method we've covered so far, from Chapter 9's embeddings through Chapter 15's re-ranking, is fundamentally about finding chunks that are *similar to the query*. That works beautifully when the answer is localized — sitting mostly inside one document, one paragraph, one table.

Relationship questions are different in kind, not just in difficulty. Consider: *"How are Acme Corp, Beta Industries, and Acme's former CFO connected?"* The answer might require combining a fact from a 2019 press release ("Beta Industries acquired a minority stake in Acme"), a fact from an SEC filing ("Jane Doe served as Acme's CFO from 2017-2021"), and a fact from a news article ("Jane Doe joined Beta Industries' board in 2022") — three documents, none of which individually mentions all three entities, and none of which is more "similar" to the query than dozens of other documents that mention Acme or Beta Industries in unrelated contexts.

Similarity search finds documents that *talk about* the entities in your query. It has no native concept of the *relationships between* those entities that span multiple documents. You can throw more chunks at the context window (Chapter 18) and hope the LLM connects the dots, but that's asking the generator to do retrieval-time reasoning that the retriever never actually did — an expensive and unreliable way to answer what is fundamentally a structural question.

> **Core idea:** some questions aren't about finding the right passage — they're about tracing a path between entities across many passages. That's a graph traversal problem wearing a retrieval costume.

---

## 22.3 What a Knowledge Graph Actually Is

A **knowledge graph**, in the RAG context, is a structured representation of your source documents as **entities** (nodes — people, companies, products, events, concepts) connected by **relationships** (edges — "acquired," "works for," "cites," "located in"). Where chunk-based retrieval stores prose, a knowledge graph stores structured triples: *(Beta Industries) --[acquired stake in]--> (Acme Corp)*.

Building this graph is itself a pipeline stage, sitting conceptually alongside the metadata extraction work from Chapter 7:

1. **Entity extraction** — identify the people, organizations, products, and other named entities in each document (often via an LLM prompted to extract structured entities, or a dedicated named-entity-recognition model)
2. **Relationship extraction** — identify how those entities relate to each other, again usually via an LLM reading each chunk and emitting structured (entity, relation, entity) triples
3. **Entity resolution** — recognizing that "Acme Corp," "Acme Corporation," and "Acme" in three different documents refer to the same node, and merging them
4. **Graph construction** — assembling the extracted triples into an actual graph structure, typically stored in a graph database (Neo4j, for example) or a graph-augmented layer on top of your existing vector store

Once built, answering a relationship question becomes a **traversal** problem instead of a similarity problem: start at the "Acme Corp" node, walk its edges, and follow the path to "Beta Industries" — picking up the supporting source chunks along the way for citation.

---

## 22.4 Community Summarization: Answering Broad, Thematic Questions

Graphs are excellent at specific traversal questions ("how is X connected to Y?"), but real users also ask broad, thematic questions that don't have a single path to follow: *"What are the major themes in our customer feedback this quarter?"* or *"What risks does this filing discuss?"* No single traversal answers these — the answer is genuinely distributed across large portions of the graph.

**Community summarization** addresses this. The idea, at a conceptual level:

1. Run a **community detection** algorithm over the graph — a clustering technique that finds tightly-connected groups of nodes (entities that relate to each other much more than they relate to the rest of the graph)
2. For each community, generate a **pre-computed summary** — an LLM reads the entities, relationships, and source text within that cluster and writes a paragraph describing what that cluster is "about"
3. At query time, a broad thematic question can be answered by retrieving and synthesizing from these **community summaries** directly, rather than trying to retrieve and reason over hundreds of raw chunks

Think of it like the difference between asking a single librarian to skim an entire library on demand versus a library that has already organized its shelves into labeled sections with a summary card on each shelf. The organizing work happens once, ahead of time (at indexing time), so that broad questions at query time are fast and coherent instead of requiring the model to synthesize dozens of scattered chunks under a token budget.

This two-tier structure — traverse the raw graph for specific relationship questions, query pre-built summaries for broad thematic questions — is the essence of what tools like Microsoft's GraphRAG popularized: retrieval that operates at multiple levels of granularity depending on what kind of question is being asked.

---

## 22.5 Hybrid Architectures: Vector Search to Find a Starting Point

In practice, very few production systems replace vector retrieval with a graph — they combine the two. The graph is excellent once you know *where to start*; it's not typically how you find that starting point in the first place.

A common pattern looks like this:

1. Run standard vector or hybrid retrieval (Chapters 9-12) against the user's query to find a small set of **seed entities** — the people, companies, or concepts the query is plausibly about
2. From those seed nodes, **traverse the graph** outward to find connected entities, relationships, and the chunks that support each edge
3. Assemble the traversal result — a small, structured neighborhood of entities and relationships, with citations — into the generation prompt, alongside or instead of raw retrieved chunks

This matters because entity resolution (Section 22.6) has to happen somewhere, and matching a user's loosely-worded query ("that CFO who left Acme") to the correctly-resolved graph node ("Jane Doe") is itself a retrieval problem — one vector search is often better suited to than exact string matching against node names. The graph then takes over for the part it's actually good at: multi-hop traversal once you're anchored to the right starting node.

This hybrid pattern also gives you a fallback: if the query doesn't resolve cleanly to graph entities at all, the system can degrade gracefully to plain chunk retrieval rather than failing outright — worth keeping in mind as a design default rather than treating graph and vector retrieval as an either/or choice.

---

## 22.6 When GraphRAG Is Worth It

GraphRAG is not a universal upgrade over chunk-based RAG — it solves a specific class of problem at a specific cost. It tends to earn its keep when:

| Signal | Favors GraphRAG |
|---|---|
| Questions frequently ask "how is X related to Y" | Yes |
| Questions ask for broad, thematic synthesis across a large corpus | Yes |
| The domain has a rich, stable web of named entities (legal, finance, research, org charts) | Yes |
| Most questions are simple factual lookups within one document | No — plain chunk retrieval is cheaper and sufficient |
| The corpus changes very frequently | Weakens the case — see Section 22.6 |
| Entities are sparse or loosely defined (casual support tickets, chat logs) | No — little relational structure to exploit |

A useful gut check: if you can already imagine drawing the answer as a small diagram with boxes and arrows, that's a strong signal the question is graph-shaped.

---

## 22.7 The Honest Cost of Building and Maintaining a Graph

This is where GraphRAG asks for real, ongoing investment, and teams underestimate it. Be clear-eyed about three costs.

**Extraction is error-prone.** Entity and relationship extraction is itself an LLM task, which means it inherits everything we've said about hallucination since Chapter 1. The model can invent a relationship that isn't actually supported by the text, miss a real one, or extract it with the wrong direction or qualifier ("acquired" versus "attempted to acquire" versus "was rumored to acquire" are very different edges). Errors compound: a wrong edge doesn't just produce one wrong answer, it can mislead every future traversal that passes through it.

**Entity resolution is genuinely hard.** Merging "Acme," "Acme Corp," and "Acme Corporation, Inc." into one node sounds simple until your corpus also contains a completely unrelated "Acme Industries" and a person also named Acme (unlikely, but the general problem — ambiguous or overlapping names — is common with people, product lines, and subsidiaries). Getting entity resolution wrong either fragments one real entity into several disconnected nodes or, worse, incorrectly merges two different entities into one, silently corrupting every downstream query.

**Keeping the graph in sync is ongoing work, not a one-time build.** Chapter 33 covers incremental indexing and data freshness for vector stores, and everything hard about that problem is *harder* for a graph. Adding a new document to a vector index just means embedding and inserting new chunks. Adding a new document to a knowledge graph means extracting its entities and relationships, resolving them against every existing node they might refer to, and potentially updating community summaries that are now stale. There is no cheap incremental update path for community summarization — a new document can, in principle, change which cluster an entity belongs to.

None of this means skip GraphRAG. It means budget for a real extraction-and-maintenance pipeline, not a one-time script, and validate the extracted graph against source documents the same way you'd validate any other model output — because it is one.

---

## 22.8 Chapter Summary

- Pure chunk-based similarity search struggles with **relationship questions** that require connecting facts scattered across many documents, because no single chunk is more "similar" to the query than any other.
- A **knowledge graph** represents documents as extracted **entities** (nodes) and **relationships** (edges), built through entity extraction, relationship extraction, entity resolution, and graph construction.
- Relationship questions become **traversal** problems on the graph instead of similarity-search problems, letting you trace paths between entities with citations back to source chunks.
- **Community summarization** clusters related entities and pre-generates a summary for each cluster, enabling broad thematic questions to be answered from summaries instead of raw chunks.
- GraphRAG earns its cost for relationship-heavy, entity-rich domains (legal, finance, research) — it's overkill for corpora dominated by simple, self-contained factual lookups.
- Building and maintaining a graph carries real, ongoing costs: **extraction is error-prone** (it's an LLM task with its own hallucination risk), **entity resolution is genuinely hard**, and **keeping the graph in sync** with a changing corpus is significantly harder than incremental vector indexing.

**Coming up next (Chapter 23):** we'll look at another way of making retrieval smarter — not by restructuring the knowledge base itself, but by teaching the RAG system to grade its own retrieval quality and correct course before generating an answer.
