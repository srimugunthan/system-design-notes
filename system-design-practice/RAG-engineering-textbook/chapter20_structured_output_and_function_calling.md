# Chapter 20: Structured Output & Function Calling in RAG

## 20.1 What This Chapter Covers

Every chapter in Part V so far has assumed the end product of generation is a paragraph of prose for a human to read. That's true for a chat interface, but it's often false everywhere else in a real system. A RAG pipeline feeding a UI widget, a downstream database, or another piece of software doesn't want a well-written paragraph — it wants a value it can parse without guessing.

This chapter covers two closely related tools for that problem: **structured output**, which constrains what the model generates to a defined schema, and **function/tool calling**, which lets the generation step trigger further actions rather than simply produce text. Together, these turn the generator from something that writes answers into something that can populate systems and take next steps — and they set up the architectural leap we'll take in Chapter 21, where the model itself starts deciding when to retrieve or call something at all.

---

## 20.2 Why Prose Isn't Always the Right Output

Everything we've covered in this book so far about grounding, citation, and context management (Chapters 17–19) has been in service of producing a good *answer*. But "good answer" means different things depending on what consumes the output.

Consider a few realistic scenarios:

- A support chatbot needs to render a **structured card** in the UI — a refund amount, a status field, a due date — not a paragraph the frontend has to parse with regex.
- A RAG system feeding a **compliance pipeline** needs each factual claim paired with a machine-readable citation, not citations embedded loosely inside prose, so the pipeline can programmatically verify every claim traces back to an approved source.
- An internal tool wants the model to **populate a form** — extract structured fields (customer name, issue category, priority) from a support ticket plus retrieved context — where any deviation from the expected shape breaks the downstream system entirely.
- A pipeline needs to **chain** the RAG answer into another automated step — schedule a follow-up, file a ticket, update a record — where "the model's answer" needs to be an unambiguous instruction, not something a human has to interpret first.

In every one of these, free-text prose is actively the wrong output format — not because it's low quality, but because nothing downstream can reliably consume it. Trying to regex-parse a natural-language answer to extract a dollar amount or a date is fragile in exactly the way that structured output generation isn't.

---

## 20.3 Constraining Generation to a Schema

Modern LLM APIs generally offer some form of structured output support, where you supply a schema — most commonly JSON Schema — and the model's output is constrained to conform to it, rather than merely being asked nicely to "please respond in JSON."

A representative schema for a RAG answer with citations:

```json
{
  "type": "object",
  "properties": {
    "answer": { "type": "string" },
    "confidence": { "type": "string", "enum": ["high", "medium", "low"] },
    "citations": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "source_id": { "type": "integer" },
          "claim": { "type": "string" }
        },
        "required": ["source_id", "claim"]
      }
    },
    "answerable": { "type": "boolean" }
  },
  "required": ["answer", "citations", "answerable"]
}
```

A model prompted with this schema, alongside the context injection and grounding instructions from Chapter 17, might return:

```json
{
  "answer": "Refunds are issued within 14 business days.",
  "confidence": "high",
  "citations": [
    { "source_id": 2, "claim": "Refunds are issued within 14 business days." }
  ],
  "answerable": true
}
```

This is worth comparing against the equivalent prose answer from Chapter 17: the information content is nearly identical, but this version is directly consumable — the `answerable` field lets a downstream system distinguish "confidently answered" from "insufficient context" without parsing natural language, and the `citations` array is exactly the machine-readable claim-to-source mapping that Chapter 19's conflict-handling and Chapter 28's faithfulness evaluation both depend on.

**How schema constraint actually works, at a conceptual level:** rather than trusting the model to freely generate text that happens to be valid JSON, many implementations restrict the model's token-by-token generation so that only tokens consistent with the schema at each step are even eligible for selection — this is often called constrained or grammar-based decoding. The practical effect is that a well-supported structured output mode makes schema violations rare rather than merely "less likely because we asked nicely." It's worth checking, per model provider, how strictly this guarantee actually holds — some implementations offer hard guarantees, others are closer to a strong nudge, and the difference matters a great deal for production reliability.

---

## 20.4 Designing Schemas That Don't Fight the Model

A schema that's too rigid can hurt answer quality in ways that aren't obvious until you see it happen. A few practical lessons:

- **Leave room for "I don't know."** A schema that requires `answer` to always be a populated string, with no `answerable: false` escape hatch, quietly re-introduces the pressure-to-guess problem from Chapter 17 — the model has nowhere to put an honest non-answer, so it fabricates one instead.
- **Don't force premature enumeration.** A citations array that requires exactly one citation per claim can push the model toward under- or over-citing just to satisfy the shape, rather than citing accurately.
- **Keep nesting shallow where possible.** Deeply nested schemas are harder for a model to fill correctly and harder for engineers to debug when they're wrong — flatter structures tend to be filled more reliably.
- **Separate "what the model is confident about" from "what it's asserting."** Fields like `confidence` or `answerable` give the model a place to express uncertainty in a structured way, instead of forcing that nuance to be smuggled into the prose of the `answer` field where nothing downstream can act on it.

> **Core idea:** structured output doesn't just reformat an answer — it forces you to decide, in advance, exactly what shape "insufficient evidence" and "partial confidence" take. A schema with no room to express those states doesn't eliminate uncertainty; it just hides it.

---

## 20.5 Function and Tool Calling: Generation That Does More Than Write

Structured output constrains what the model *writes*. Function calling (also called tool calling, or tool use) goes a step further: it lets the model's output be an *instruction to invoke something* — an API, a database query, a calculator — with the results potentially fed back to the model before it produces a final answer.

The mechanics, at a conceptual level:

1. The model is given a set of available tools, each described by a name, a description, and a parameter schema (structurally very similar to the JSON Schema from 20.3).
2. Given the user's question and the retrieved context, the model can respond not with a final answer, but with a request to call a specific tool with specific arguments.
3. The calling application executes that tool call outside the model (this is a real API call, database query, etc. — the model itself does not execute anything) and returns the result.
4. The model incorporates that result — alongside whatever was originally retrieved — into its next step, which might be another tool call or a final answer.

Where this intersects directly with RAG: retrieval itself doesn't have to be a fixed, upfront step anymore. Instead of always running a vector search before generation begins, the model can be given a `search_knowledge_base` tool and decide, based on the question, whether retrieval is even necessary, what to search for, and whether one search is enough. A few illustrative examples of tools that pair naturally with a RAG system:

| Tool | What it does | Why it's useful alongside retrieval |
|---|---|---|
| `search_knowledge_base(query)` | Runs the retriever with a model-chosen query | Lets the model reformulate or refine what it searches for |
| `get_live_account_status(account_id)` | Calls a live internal API | Some facts (today's account balance) are never going to live in a static document index — they need a live call instead of retrieval |
| `lookup_exchange_rate(currency_pair)` | Calls an external service | Retrieval is the wrong tool for information that changes by the minute |
| `file_support_ticket(details)` | Triggers a downstream action | Lets the answer step end in an action, not just text |

That third row is worth sitting with: it's a direct, concrete answer to a limitation raised all the way back in Chapter 1 — a static knowledge base, however well-indexed, can never hold information that changes faster than it's re-ingested. Function calling gives the generation step a way to reach past the index entirely and get the answer live, when that's genuinely the right tool for the fact being asked about.

---

## 20.6 Structured Output and Function Calling Working Together

In practice these two tools are frequently combined rather than used separately. A common pattern in a mature RAG system:

1. The model receives the question and decides — via a tool call — whether it needs to search the knowledge base, call a live API, or both.
2. Results come back and are added to the model's working context.
3. The model may issue further tool calls if the first round of results was insufficient (a preview of the multi-hop and iterative patterns from Chapter 16, now driven by the model's own tool-calling decisions rather than a fixed pipeline).
4. Once enough evidence is gathered, the model produces a **final structured output** — not prose — conforming to the schema the application expects, complete with citations tied back to whichever sources (retrieved documents or live tool results) actually informed each claim.

This combination is effectively a preview of **Agentic RAG**, the subject of Chapter 21: once the model can decide *whether* and *what* to retrieve or call, rather than always operating on a fixed, pre-fetched context block, retrieval stops being a rigid upfront pipeline stage and becomes one option among several tools the model reaches for as needed.

---

## 20.7 Where Structured Output and Function Calling Fall Short

As with every technique in this book, it's worth being direct about what these tools don't solve:

- **Schema conformance is not the same as correctness.** A perfectly valid JSON object can still contain a hallucinated `answer` field or a citation that doesn't actually support its claim. Structured output makes errors easier to *detect* programmatically; it doesn't prevent them.
- **Function calling adds a new failure surface.** A model can call the wrong tool, pass malformed or hallucinated arguments (calling `lookup_exchange_rate` with a currency pair that doesn't exist, for instance), or call a tool unnecessarily when the retrieved context already had the answer. Each of these needs to be handled by the calling application, not assumed away.
- **More tool-calling rounds mean more latency and cost.** Every round trip — model decides to call a tool, application executes it, result goes back to the model — adds real time and real token cost, echoing the same tradeoffs raised in Chapter 18 about context growth.
- **Giving a model the ability to trigger real actions raises the stakes on grounding and guardrails considerably.** A hallucinated fact in a chat answer is bad; a hallucinated argument to a `file_support_ticket` or a live financial API call is a different category of problem entirely, one that Chapter 34's security and guardrails material addresses directly.

---

## 20.8 Chapter Summary

- Production RAG systems frequently need **structured, machine-parseable output** — for UI rendering, downstream systems, or programmatic citation verification — rather than free-text prose.
- **Schema-constrained generation** (typically via JSON Schema) restricts the model's output shape, often through constrained decoding, making schema violations rare rather than merely discouraged — though the strength of this guarantee varies by provider.
- Good schema design **leaves explicit room for uncertainty** (`answerable: false`, confidence fields) — a rigid schema with no way to express "I don't know" reintroduces the pressure-to-guess problem from Chapter 17.
- **Function/tool calling** lets the model's output be a request to invoke a real action — a search, a live API call, a downstream task — executed outside the model and fed back into its context.
- Combining retrieval with live tool calls addresses information that a static index fundamentally cannot hold, such as data that changes faster than re-ingestion cycles.
- Structured output and function calling together are the architectural seed of **Agentic RAG** (Chapter 21), where the model decides when and what to retrieve or call rather than following a fixed pipeline.
- Neither technique guarantees correctness — schema conformance is not truth, and tool calls can be malformed or misdirected — which raises the stakes on grounding, evaluation, and guardrails rather than lowering them.

**Coming up next (Chapter 21):** we open Part VI by letting the model take the wheel — Agentic RAG, where retrieval and tool use become decisions the model makes dynamically rather than a fixed sequence of pipeline steps.
