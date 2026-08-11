# Chapter 13: Query Understanding

## 13.1 What This Chapter Covers

Here's an uncomfortable truth we need to confront early in Part IV: **the sentence a user types is often a bad search query.** People type questions the way they'd ask a colleague, not the way a search system needs them phrased. So before we can retrieve anything well, we need to fix the query itself.

This chapter covers the discipline of **query understanding** — rewriting, expanding, decomposing, and even reimagining the user's question before it ever touches your retriever. Get this step wrong, and no amount of clever indexing (Part III) or re-ranking (Chapter 15) will save you, because you'll be searching for the wrong thing very efficiently.

---

## 13.2 Why the Raw Query Is Often the Wrong Query

Imagine a user typing this into your RAG system:

> "what about the refund thing from before, does it still apply"

A human colleague, with shared context, could probably figure out what this means. A retriever comparing this sentence's embedding (Chapter 9) against your document store has a much harder time. This single sentence has several problems stacked on top of each other:

- **Too short and vague** — "the refund thing" doesn't name a policy, product, or timeframe
- **Conversational, not informational** — it's phrased as a follow-up in a dialogue, not a standalone statement of information need
- **Missing key terms** — the actual policy might be called "Return and Refund Policy v3," a phrase the user never used
- **Ambiguous references** — "before" and "still" imply context from earlier in a conversation that the retriever has no access to unless it's explicitly carried forward

Contrast that with a query like "What is the current refund policy for orders placed within 30 days?" — specific, self-contained, and full of terms that are likely to appear verbatim in the right document. That's the gap query understanding exists to close.

**Key idea:**

> Retrieval systems are much better at matching *statements* than *questions*, and much better at matching *specific* language than *vague* language. Query understanding is the step where we transform what the user asked into something more retrievable — without changing what they meant.

This matters more as RAG systems move from single-turn demos into multi-turn assistants, customer support tools (Chapter 36), and agentic workflows (Chapter 21), where queries increasingly arrive stripped of context, shorthand, or mid-conversation.

---

## 13.3 Query Rewriting

**Query rewriting** is the simplest form of query understanding: take the user's raw input and reformulate it into a cleaner, more explicit, more searchable version — while preserving intent.

Common rewriting tasks include:

| Problem in raw query | What rewriting does |
|---|---|
| Conversational shorthand ("does it still apply") | Resolves pronouns and references using conversation history |
| Spelling errors or informal phrasing | Normalizes to standard terminology |
| Missing entities (mentioned earlier in chat) | Injects the resolved entity explicitly ("the refund thing" → "the 30-day refund policy") |
| Overly broad questions | Narrows scope based on available context (user's account type, product line, etc.) |

In practice, rewriting is usually done by handing the LLM the conversation history and the latest message, and asking it to produce a single, self-contained search query. This is a small, cheap LLM call, but it pays for itself: a well-formed query dramatically increases the odds that the embedding step and the retriever land near the right documents.

It's worth being honest about a failure mode here too: an overly aggressive rewrite can *drift* from what the user actually meant, especially when the LLM guesses at ambiguous references. A good rewriting prompt is conservative — it clarifies, it doesn't invent.

---

## 13.4 Query Expansion

Where rewriting cleans up *one* query, **query expansion** produces *more* terms — synonyms, related phrases, and alternate wordings — to increase the odds of matching documents that use different vocabulary than the user did.

This matters because your documents and your users rarely use identical language. A user might ask about "canceling my plan," while your knowledge base only ever refers to "subscription termination." A pure similarity search can survive small vocabulary gaps because embeddings capture meaning, not just words — but keyword-based and hybrid search (Chapter 12) are much more literal, and even embeddings benefit from having more surface area to match against.

Expansion typically looks like generating a small set of related terms or reformulated queries, for example:

- Original: "how do I cancel my plan"
- Expanded terms: "subscription termination," "cancel subscription," "close account," "stop billing"

These expanded terms can be used to broaden a keyword search, added as additional embedding queries (more on this pattern as *multi-query retrieval* in Chapter 14), or blended into a single richer query. The tradeoff is real: expand too aggressively and you risk pulling in loosely related, lower-precision documents that dilute your candidate set. Expansion is a dial, not a switch — most production systems tune how many expansion terms to generate and how much weight to give them.

---

## 13.5 Query Decomposition

Some questions aren't hard because they're vague — they're hard because they're actually *several questions wearing a trench coat.*

Consider: "How does our return policy compare to our main competitor's, and did either of us change it in the last year?" A single retrieval pass against this compound question will struggle, because no single document is likely to answer all of it — and the embedding of the whole sentence will be a blurry average of several distinct information needs.

**Query decomposition** solves this by breaking a complex, multi-part question into a set of simpler, independent sub-questions, each of which can be retrieved for separately:

1. "What is our current return policy?"
2. "What is our competitor's current return policy?"
3. "Did our return policy change in the last 12 months?"
4. "Did our competitor's return policy change in the last 12 months?"

Each sub-question runs through retrieval on its own, and the results are combined before generation. This tends to produce noticeably better answers on genuinely compound questions, at the cost of doing multiple retrieval passes instead of one.

Decomposition is closely related to — and often the entry point into — the deeper topic of **multi-hop retrieval**, which we cover fully in Chapter 16. The distinction is one of degree: decomposition here typically means splitting a question we can already see is compound, up front, into independent parallel sub-questions. Multi-hop retrieval goes further, handling cases where later sub-questions *depend* on the answers to earlier ones, discovered only as retrieval proceeds.

---

## 13.6 HyDE: Hypothetical Document Embeddings

Now for the most counter-intuitive technique in this chapter — one that sounds like it shouldn't work, and yet often does.

Recall from Chapter 9 that embedding-based retrieval works by comparing the embedding of the query to the embeddings of document chunks, and returning the chunks whose embeddings are closest. Here's the problem: a *question* and its *answer* often live in noticeably different regions of embedding space, even when the answer is exactly what the question is looking for. "What causes inflation?" is phrased very differently, structurally and lexically, from a textbook paragraph that actually explains the causes of inflation — even though the paragraph is precisely the right retrieval target.

**HyDE (Hypothetical Document Embeddings)** works around this mismatch with a clever trick:

> Instead of embedding the user's question directly, first ask the LLM to write a *hypothetical answer* to the question — a plausible-sounding passage, even though the LLM may not know if it's factually correct — and then embed *that hypothetical answer* instead of the original question. Use this embedding to search the document store.

Why does this work? Because a hypothetical answer, even a fabricated or partially wrong one, tends to be phrased much more like a real document than the original question is. It uses declarative sentences, domain terminology, and the same "shape" of language as the documents you're searching over. Its embedding therefore tends to land closer, in vector space, to real answer-bearing documents than the bare question's embedding would.

A simplified walkthrough:

```
User question: "What causes inflation?"
      │
      ▼
[1] LLM generates a hypothetical answer (may contain errors — that's fine)
      "Inflation is caused by an increase in the money supply relative to
       goods and services, demand outpacing supply, and rising production
       costs that get passed on to consumers..."
      │
      ▼
[2] Embed the hypothetical answer (not the original question)
      │
      ▼
[3] Search the vector store using this embedding
      │
      ▼
[4] Retrieve real documents that are semantically close to this passage
```

Note the subtlety: the hypothetical answer's *content* doesn't need to be correct — it's never shown to the user, and it's discarded after retrieval. It only needs to be *shaped like* a real answer, so its embedding lands in the right neighborhood. The actual answer the user sees is still generated later, grounded in the real documents retrieved this way (Part V).

HyDE tends to help most on queries where the vocabulary gap between question and answer is large — technical or specialized domains, for instance — and helps less on queries that are already close to document language. It also adds an extra LLM call before retrieval even starts, which is a latency and cost tradeoff worth weighing against the accuracy gain, a theme we'll return to repeatedly starting in Chapter 14.

---

## 13.7 Combining These Techniques

These four techniques are not mutually exclusive — production systems frequently chain them:

| Technique | Solves | Typical cost |
|---|---|---|
| **Rewriting** | Vague, conversational, or context-dependent queries | One small LLM call |
| **Expansion** | Vocabulary mismatch between user and documents | One small LLM call (or a lookup table) |
| **Decomposition** | Compound, multi-part questions | Multiple retrieval passes |
| **HyDE** | Structural mismatch between question phrasing and answer phrasing | One LLM call + embedding |

A realistic pipeline might rewrite a conversational query into a self-contained one, decompose it into sub-questions if it's compound, and run HyDE on each sub-question before retrieval. Every added step is also an added point of latency and potential drift, so most teams introduce these incrementally, guided by evaluation (Part VII) rather than applying all four unconditionally to every query.

---

## 13.8 A Note of Honesty: Query Understanding Is Not Mind-Reading

It's tempting to think that with enough rewriting, expansion, and clever tricks, we can always figure out exactly what the user meant. We can't, fully. A few real limits worth internalizing:

- **Ambiguity sometimes requires clarification, not inference.** If a user asks "what's the status," and there are three plausible things they could mean, guessing wrong confidently is often worse than asking a follow-up question.
- **Rewriting can silently change intent.** An LLM resolving "it" to the wrong prior entity produces a well-formed, confident, *wrong* query — and everything downstream inherits that mistake.
- **These techniques add latency and cost.** Every extra LLM call before retrieval is time the user is waiting and money you're spending, and in latency-sensitive applications this budget is not free.
- **None of this fixes a bad index.** Query understanding can only help you ask a better question of the documents you have. It can't retrieve information that was never indexed in the first place (a problem we return to in Chapters 5–8).

Query understanding raises the *ceiling* of what good retrieval can achieve. It doesn't guarantee you'll hit it.

---

## 13.9 Chapter Summary

- Raw user queries are frequently poor search queries — too short, conversational, ambiguous, or missing key terminology.
- **Query rewriting** reformulates a query into a clean, self-contained, explicit form, often resolving conversational references using chat history.
- **Query expansion** adds synonyms and related terms to bridge vocabulary gaps between how users ask and how documents are written.
- **Query decomposition** splits compound, multi-part questions into independent sub-questions that can each be retrieved for separately.
- **HyDE (Hypothetical Document Embeddings)** has the LLM generate a plausible hypothetical answer first, then embeds *that answer* — rather than the original question — because answer-shaped text tends to land closer, in embedding space, to real answer-bearing documents.
- These techniques can be combined, but each adds latency and cost, and each introduces a new opportunity to drift from the user's true intent.
- Query understanding improves what retrieval is capable of finding — it cannot compensate for a poorly built index or genuinely ambiguous questions.

**Coming up next (Chapter 14):** with a well-formed query in hand, we turn to the retriever itself — comparing top-k retrieval, MMR, multi-query retrieval, and parent-document retrieval as strategies for turning a good query into a genuinely useful set of retrieved chunks.
