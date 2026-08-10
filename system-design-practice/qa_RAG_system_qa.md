# Handling Context Limit Overflow in RAG Systems

**How do you handle cases where the retrieved documents exceed the LLM's context limit?**

When retrieved documents exceed the context window, there are several complementary strategies, typically applied in combination:

## 1. Reduce What You Retrieve (Pre-context)

**Tighter retrieval (top-k tuning)**
- Lower `k` (number of chunks retrieved) — often the simplest fix
- Use similarity score thresholds to drop low-relevance chunks rather than always returning a fixed k

**Smaller, denser chunks**
- Smaller chunk sizes (e.g., 256–512 tokens instead of 1000+) pack more distinct information per token
- Use semantic chunking (split on topic/section boundaries) instead of fixed-size splitting, so each chunk carries more signal per token

## 2. Rerank and Prune (Post-retrieval, pre-context)

- Retrieve a larger candidate pool (e.g., top-50) with a cheap retriever, then use a cross-encoder reranker (e.g., Cohere rerank, BGE-reranker) to select the true top-k that fits the budget
- This decouples "recall" (cast a wide net) from "precision" (only pass what matters into context)

## 3. Compress Before Passing to the LLM

- **Extractive compression**: pull only the sentences/passages most relevant to the query out of each chunk (e.g., LLMLingua, or a lightweight extractive summarizer)
- **Contextual compression retrievers** (LangChain has this pattern): retrieve → compress each doc down to just the query-relevant parts → then assemble
- **Map-reduce summarization**: summarize each chunk individually (map), then summarize the summaries (reduce) if you're synthesizing across many documents rather than doing precise QA

## 4. Iterative / Multi-Pass Strategies

- **Refine chain**: process documents sequentially, updating a running answer with each new chunk, rather than loading everything at once
- **Map-rerank**: run the query against each chunk independently, get an answer + confidence score per chunk, then pick the best-scoring answer
- **Hierarchical retrieval**: first retrieve at a coarse level (document/section summaries), narrow down to which documents matter, then retrieve fine-grained chunks only from those

## 5. Architectural Approaches

- **Long-context models**: if the use case allows it, route to a long-context model (Claude, Gemini) instead of engineering around a small window — often cheaper in engineering time even if more expensive per call
- **RAG-Fusion / query decomposition**: break the user query into sub-queries, retrieve smaller focused sets per sub-query, and only merge the most relevant results
- **Memory/caching layers**: for repeated queries over the same corpus, cache summarized representations rather than re-processing raw chunks each time

## Practical Recommendation

For most production RAG systems (this maps well to something like your AuditAgent pipeline), a reasonable default stack is:

1. Retrieve top-30–50 via vector search (cheap, high recall)
2. Rerank down to top-5–8 via a cross-encoder
3. Apply contextual compression to strip irrelevant sentences from each surviving chunk
4. If the corpus is very heterogeneous (e.g., audit logs + policy docs + prior findings), add a hierarchical pre-filter so you're not reranking across totally unrelated document types

The key tradeoff to watch is recall vs. precision — overly aggressive compression/reranking risks dropping the one chunk with the actual answer, which is especially costly in an audit or fraud-detection context where false negatives from missing evidence matter more than in general Q&A.

--
**what is semantic caching**

This video explores strategies for implementing **caching** in applications that use **LLMs** to manage repeated prompts and reduce latency or costs. The discussion covers two primary caching methods:

* **Exact Query Caching (0:39 - 2:04):** This is the most basic form of caching, where identical user prompts are stored in a key-value cache. If a request is repeated, the system retrieves the previous response from the cache instead of querying the LLM again. Techniques like *time-to-live* (TTL) or *Least Recently Used* (LRU) are suggested to prevent serving stale data.
* **Semantic Caching (2:04 - 4:14):** When prompts are not identical but share the same meaning (semantic similarity), a semantic cache is used. This involves converting queries into **vectors** and performing *vector similarity searches* (like cosine similarity). If the similarity score is above a threshold (e.g., >0.9), the system returns the cached response for the related query.

**Trade-offs (4:14 - 5:52):**
* **Accuracy:** Semantic caching may result in inexact answers if the system incorrectly identifies two semantically close but contextually different prompts as a match.
* **Latency:** Vector searching for semantic matches is generally slower than an exact key-value hashmap lookup.
* **Cost:** While caching is generally efficient, the benefit is primarily in avoiding redundant and expensive LLM calls, as the *embedding* models used for vectorization are typically much cheaper and faster to run.