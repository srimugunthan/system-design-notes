# Chapter 21: Agentic RAG

## 21.1 What This Chapter Covers

Every RAG system we've built so far follows the same shape: retrieve once, generate once. The user asks a question, the retriever fetches some chunks, the model writes an answer, and the pipeline is done. But what happens when a single retrieval pass genuinely isn't enough — when the model needs to search, look at what came back, realize it's insufficient, search again with a different query, and only then answer?

That's the question this chapter answers: **what changes when we let the model itself decide whether to retrieve, what to search for, and when it has enough information to stop?**

---

## 21.2 The Fixed Pipeline's Ceiling

Chapter 16 introduced multi-hop retrieval — decomposing a complex question into sub-questions and retrieving for each one. That was already a step beyond "retrieve once, generate once." But even multi-hop retrieval, as we described it, is typically still a *fixed* pipeline: a predetermined number of hops, a predetermined decomposition strategy, executed the same way regardless of how the retrieved evidence actually looks along the way.

Fixed pipelines are predictable, testable, and cheap to reason about. They're also brittle in a specific way: they cannot adapt mid-execution. If hop one returns garbage, a fixed two-hop pipeline still dutifully executes hop two using a plan built on that garbage. If a question turns out to need four searches instead of two, a fixed pipeline simply doesn't do the other two.

**Agentic RAG** is the response to this brittleness. Instead of hard-coding the retrieval plan in advance, we give the model tools — a search function, maybe a filter function, maybe a calculator — and let it decide, step by step, what to do next based on what it has seen so far.

> **Core idea:** in agentic RAG, retrieval is not a fixed stage in a pipeline — it is one of several actions the model can choose to take, as many times as it decides it needs to, in whatever order the problem demands.

---

## 21.3 From Pipeline to Loop

Picture the difference as two different kinds of employees.

The fixed-pipeline system is like a new hire following a laminated checklist: "Step 1, search the database. Step 2, write a summary." They follow it exactly, every time, whether or not it makes sense for the specific request in front of them.

The agentic system is like an experienced analyst. Given a question, they think for a moment about what they'd need to know, run a search, skim the results, and — critically — *decide* what to do next. Maybe the results answer the question and they write it up. Maybe the results are irrelevant and they try a different search term. Maybe the question actually splits into three smaller questions they hadn't noticed at first glance.

This is a loop, not a pipeline: **reason, act, observe, repeat** — until the model decides it has enough to answer, or it hits a limit we've imposed (more on that safety net in Section 21.6).

---

## 21.4 The ReAct Pattern

The most common way to implement this loop is a pattern called **ReAct**, short for *Reasoning and Acting*. The idea, built directly on the function-calling machinery from Chapter 20, is to interleave two kinds of output from the model:

- **Thought** — a short piece of reasoning about what to do next ("The user is asking about a policy that may have changed recently; I should search for the most recent version rather than assume the first result is current.")
- **Action** — a concrete tool call, such as `search("refund policy 2026 update")`, along with its arguments

After each action, the system executes the tool (runs the retrieval call, hits the API, queries the database) and feeds the result back to the model as an **Observation**. The model then produces another Thought, decides on another Action — or decides it's done and produces a final answer.

```
Thought:  I need the current refund policy, not the general one.
Action:   search("refund policy exceptions 2026")
Observation: [3 chunks returned, none mention exceptions]
Thought:  These results are too general. Let me narrow the query.
Action:   search("refund policy exceptions holiday season")
Observation: [chunk found: exceptions apply Nov 15 - Jan 5]
Thought:  This answers the question. I can respond now.
Final Answer: ...
```

Nothing here is exotic — it's the same request/response loop from Chapter 20's function calling, just repeated in a cycle where the model, not a hard-coded controller, decides how many cycles to run and what each one searches for.

---

## 21.5 Self-Querying: Turning Language into Structured Filters

One specific and very useful skill to give an agentic retriever is **self-querying**: teaching the model to translate a natural-language question into a structured, filtered retrieval query against the metadata fields we built in Chapter 7.

Consider the question: *"What did our EMEA sales team say about churn risk in Q4 last year?"* A naive retriever just embeds this whole sentence and does a similarity search. A self-querying retriever instead recognizes that this sentence contains a mix of a semantic part and several structured constraints, and produces something closer to:

```json
{
  "semantic_query": "churn risk discussion",
  "filters": {
    "region": "EMEA",
    "department": "sales",
    "quarter": "Q4",
    "year": 2025
  }
}
```

That structured query is then run against the vector store's metadata filters (Chapter 10) alongside the semantic similarity search, dramatically narrowing the candidate set before similarity even matters. This is often the single highest-leverage upgrade you can make to a retriever handling questions with dates, departments, product names, or other structured attributes buried in ordinary language — it turns an implicit constraint into an explicit one instead of hoping the embedding model captures "EMEA" and "Q4" as strongly as it captures "churn."

Self-querying is a tool the agent can invoke as part of its loop: reason about the question, decide it has filterable structure, construct the filtered query, retrieve, observe, and continue.

---

## 21.6 Giving the Agent Judgment: "Is This Enough?"

The genuinely new capability agentic RAG introduces — beyond just "search more than once" — is letting the model judge the *sufficiency* of what it has retrieved. A fixed pipeline has no concept of "enough." It runs its retrieval step and moves on regardless of quality. An agent, in principle, can look at its observations and reason: *these three chunks don't actually answer the question — I should search again with different terms*, or conversely, *this fully answers it — no need to keep searching.*

This sufficiency judgment is powerful, but it is also the least reliable part of the whole design, because we are now trusting the model's self-assessment rather than an external check. Chapter 23 covers this in much more depth — Corrective RAG and Self-RAG are, in effect, formalized, more disciplined versions of exactly this "is this good enough?" judgment, with explicit grading steps rather than an implicit one folded into free-form reasoning.

Because an ungoverned loop can, in theory, keep searching forever, real systems impose guardrails:

- A **maximum number of iterations** (e.g., stop after 4-6 tool calls regardless of what the model wants)
- A **maximum tool-call budget** tied to cost or latency limits
- A **timeout** that forces a "best effort" answer using whatever has been retrieved so far
- Sometimes a separate, cheaper model or heuristic that vetoes obviously unproductive loops (repeating a near-identical query, for instance)

None of these guardrails are optional in production. An agent with an open-ended loop and no limits is a cost and latency incident waiting to happen.

---

## 21.7 The Honest Tradeoffs

Agentic RAG is genuinely more capable than a fixed pipeline for questions that need adaptive, multi-step investigation. It is also meaningfully harder to operate. Be honest with yourself about what you're giving up.

| Dimension | Fixed Pipeline | Agentic RAG |
|---|---|---|
| **Latency** | Predictable, bounded | Variable — depends on how many loops the model runs |
| **Cost** | One retrieval + one generation call | Multiple LLM calls per query (reasoning steps add up) |
| **Predictability** | Same steps every time | Behavior can vary run to run, even for similar questions |
| **Debuggability** | Easy to trace — one path | Harder — need to log the whole reasoning/action trace |
| **Capability ceiling** | Bounded by the fixed plan | Can adapt to genuinely open-ended questions |
| **Failure mode** | Under-retrieves and answers anyway | Can loop unproductively, over-search, or stop too early |

The latency and cost columns deserve emphasis: every reasoning step and every tool call is a round trip to the LLM, and those add up. A question that would cost one retrieval and one generation call in a fixed pipeline might cost four to eight LLM calls in an agentic loop. For high-volume, low-complexity queries — the "what's our return policy" kind of question — that overhead is usually not worth paying. Agentic RAG earns its cost on genuinely complex, multi-part, or ambiguous questions; it's rarely the right default for everything.

A practical middle ground many teams land on: route simple queries to a fixed pipeline and reserve the agentic loop for questions a lightweight classifier (or the model itself) flags as needing multi-step investigation. That routing decision is itself worth engineering carefully — treat "should this even be agentic?" as a first-class design question rather than making every query pay the agentic tax by default.

---

## 21.8 Agentic RAG Is Not Autonomy Without Limits

It's tempting to think of agentic RAG as "the model can now do anything it needs to." In practice, an agent is only as good as the tools it's given and the judgment it exercises about when to use them — and that judgment is itself produced by an LLM, with all the unreliability we've discussed since Chapter 1. An agentic retriever can:

- Search with the wrong terms repeatedly without noticing the pattern
- Judge insufficient evidence as "good enough" and answer anyway
- Rack up tool calls on a question a fixed pipeline would have answered in one hop
- Behave differently on two nearly identical questions, making regression testing harder

None of this means agentic RAG isn't worth building — for the right class of questions, it clearly is. It means treating it as a capability with real operational cost, not a free upgrade you bolt onto every pipeline.

---

## 21.9 Chapter Summary

- **Agentic RAG** replaces a fixed "retrieve once, generate once" pipeline with a loop where the model decides whether to retrieve, what to search for, and when to stop.
- The **ReAct pattern** (Reasoning + Acting) interleaves short reasoning steps with tool-call actions and observations, built on the function-calling foundation from Chapter 20.
- **Self-querying** teaches the model to translate natural-language questions into structured, filtered queries against metadata (Chapter 7), separating semantic search from explicit filters like date or department.
- The key new capability agentic RAG introduces is letting the model judge **retrieval sufficiency** — whether it has enough evidence to answer — a judgment formalized further by Corrective RAG and Self-RAG in Chapter 23.
- Agentic loops need explicit **guardrails**: iteration caps, tool-call budgets, and timeouts, or they risk unbounded cost and latency.
- Compared to fixed pipelines, agentic RAG trades **predictability, cost, and debuggability** for adaptability — a tradeoff worth making for complex, multi-part questions, not for every query by default.

**Coming up next (Chapter 22):** we'll look at another way to go beyond flat chunk retrieval — representing knowledge as a graph of entities and relationships, and how GraphRAG answers questions that pure similarity search structurally cannot.
