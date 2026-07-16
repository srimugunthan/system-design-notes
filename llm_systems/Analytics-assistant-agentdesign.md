# System Design: Production-Grade Multi-Agent Analytics Assistant with LangGraph

> Enterprise-grade design for an analytics assistant that routes queries intelligently across RAG, SQL execution, API tools, and direct LLM response — with verification loops, observability, and horizontal scalability.

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Agent Graph Design](#3-agent-graph-design)
4. [Node Specifications](#4-node-specifications)
5. [Routing Logic](#5-routing-logic)
6. [RAG Pipeline](#6-rag-pipeline)
7. [Tool Execution Layer](#7-tool-execution-layer)
8. [Verification & Correction Loops](#8-verification--correction-loops)
9. [State Management](#9-state-management)
10. [Observability Stack](#10-observability-stack)
11. [Scalability & Deployment](#11-scalability--deployment)
12. [Tech Stack Summary](#12-tech-stack-summary)
13. [Trade-off Analysis](#13-trade-off-analysis)
14. [Failure Modes & Mitigations](#14-failure-modes--mitigations)

---

## 1. Problem Statement & Scope

### What We Are Building

An enterprise analytics assistant that accepts natural language queries and routes them to the most appropriate execution strategy — retrieval from a document corpus, SQL execution against a data warehouse, external API calls, or a direct LLM response — before verifying and returning a structured answer.

### Core Requirements

- **Functional:** Handle heterogeneous query types (definitional, analytical, operational, conversational)
- **Correctness:** Verify answers before returning them; loop back and self-correct on failure
- **Latency:** p50 < 3s for direct LLM, p50 < 8s for RAG/SQL, p99 < 30s for complex multi-hop
- **Observability:** Every node transition, tool call, token count, and latency must be traceable
- **Scalability:** Handle 1,000+ concurrent sessions without state collision

### Out of Scope

- Fine-tuning the underlying LLM
- Real-time streaming data ingestion (batch ingestion only)
- Multi-modal inputs (text only in v1)

---

## 2. High-Level Architecture

```
                          ┌─────────────────────────────────────────────────┐
                          │                  Client Layer                   │
                          │     REST API  /  WebSocket  /  Slack Bot         │
                          └───────────────────┬─────────────────────────────┘
                                              │
                          ┌───────────────────▼─────────────────────────────┐
                          │              API Gateway (FastAPI)               │
                          │   Auth · Rate Limiting · Session Hydration       │
                          └───────────────────┬─────────────────────────────┘
                                              │
                          ┌───────────────────▼─────────────────────────────┐
                          │           LangGraph Orchestration Engine         │
                          │                                                  │
                          │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
                          │  │  Router  │→ │  RAG     │  │  SQL Agent   │  │
                          │  │  Node    │  │  Agent   │  │              │  │
                          │  └──────────┘  └──────────┘  └──────────────┘  │
                          │       │                                          │
                          │  ┌────▼─────┐  ┌──────────┐  ┌──────────────┐  │
                          │  │  Tool    │  │ Verifier │  │  Synthesizer │  │
                          │  │  Agent   │  │  Node    │  │  Node        │  │
                          │  └──────────┘  └──────────┘  └──────────────┘  │
                          └───────────────────┬─────────────────────────────┘
                                              │
              ┌───────────────────────────────┼────────────────────────────┐
              │                               │                            │
   ┌──────────▼──────────┐      ┌─────────────▼───────────┐   ┌───────────▼─────────┐
   │   Vector Store       │      │   Data Warehouse        │   │  External APIs      │
   │   (Weaviate)         │      │   (Snowflake / BigQuery) │   │  (REST / GraphQL)   │
   └─────────────────────┘      └─────────────────────────┘   └─────────────────────┘
              │                               │                            │
              └───────────────────────────────┼────────────────────────────┘
                                              │
                          ┌───────────────────▼─────────────────────────────┐
                          │              Observability Stack                 │
                          │   LangSmith · Prometheus · Grafana · Jaeger      │
                          └─────────────────────────────────────────────────┘
```

---

## 3. Agent Graph Design

The system is modeled as a **directed graph with conditional edges** in LangGraph. Each node is a stateless function that reads from and writes to a shared `AnalyticsState` object.

```
                    ┌─────────────┐
                    │    START    │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   ROUTER    │  ← Classifies query intent
                    └──────┬──────┘
                           │
           ┌───────────────┼──────────────────┐
           │               │                  │
    ┌──────▼──────┐  ┌─────▼──────┐   ┌──────▼──────┐
    │  RAG AGENT  │  │ SQL AGENT  │   │ TOOL AGENT  │
    └──────┬──────┘  └─────┬──────┘   └──────┬──────┘
           │               │                  │
           └───────────────┼──────────────────┘
                           │
                    ┌──────▼──────┐
                    │  VERIFIER   │  ← Checks answer quality
                    └──────┬──────┘
                           │
               ┌───────────┴───────────┐
               │                       │
        ┌──────▼──────┐        ┌───────▼──────┐
        │  CORRECTOR  │        │  SYNTHESIZER │
        │  (loop back)│        │  (finalize)  │
        └──────┬──────┘        └───────┬──────┘
               │                       │
               └───────────┬───────────┘
                           │
                    ┌──────▼──────┐
                    │     END     │
                    └─────────────┘
```

### Graph Definition (Code Sketch)

```python
from langgraph.graph import StateGraph, END
from state import AnalyticsState

def build_graph() -> StateGraph:
    graph = StateGraph(AnalyticsState)

    graph.add_node("router",      router_node)
    graph.add_node("rag_agent",   rag_agent_node)
    graph.add_node("sql_agent",   sql_agent_node)
    graph.add_node("tool_agent",  tool_agent_node)
    graph.add_node("verifier",    verifier_node)
    graph.add_node("corrector",   corrector_node)
    graph.add_node("synthesizer", synthesizer_node)

    graph.set_entry_point("router")

    graph.add_conditional_edges("router", route_decision, {
        "rag":    "rag_agent",
        "sql":    "sql_agent",
        "tool":   "tool_agent",
        "direct": "synthesizer",
    })

    for agent in ["rag_agent", "sql_agent", "tool_agent"]:
        graph.add_edge(agent, "verifier")

    graph.add_conditional_edges("verifier", verify_decision, {
        "pass":   "synthesizer",
        "retry":  "corrector",
        "fail":   "synthesizer",   # Fail gracefully with explanation
    })

    graph.add_conditional_edges("corrector", correction_route, {
        "rag":  "rag_agent",
        "sql":  "sql_agent",
        "tool": "tool_agent",
    })

    graph.add_edge("synthesizer", END)

    return graph.compile(checkpointer=RedisCheckpointer())
```

---

## 4. Node Specifications

### 4.1 Router Node

**Responsibility:** Classify the incoming query into one of four strategies.

**Implementation:** A lightweight LLM call with a structured output schema. Uses a small, fast model (GPT-4o-mini or Haiku) to keep routing latency under 300ms.

```python
class RouterDecision(BaseModel):
    strategy: Literal["rag", "sql", "tool", "direct"]
    confidence: float
    reasoning: str
    sub_queries: list[str]  # For multi-hop decomposition

async def router_node(state: AnalyticsState) -> AnalyticsState:
    decision = await llm_structured(
        model="gpt-4o-mini",
        system=ROUTER_SYSTEM_PROMPT,
        user=state.query,
        output_schema=RouterDecision
    )
    return state.update(
        strategy=decision.strategy,
        sub_queries=decision.sub_queries,
        routing_confidence=decision.confidence
    )
```

**Routing heuristics in the system prompt:**

- `rag` → "explain", "what is", "how does", "according to", document-heavy queries
- `sql` → "how many", "total", "average", "trend", "compare", data-heavy queries
- `tool` → "current price", "weather", "fetch", "latest", real-time data queries
- `direct` → greetings, simple math, conversational follow-ups

### 4.2 RAG Agent Node

**Responsibility:** Retrieve relevant document chunks and synthesize an answer.

```python
async def rag_agent_node(state: AnalyticsState) -> AnalyticsState:
    # Step 1: Query expansion
    expanded_queries = await expand_query(state.query, state.conversation_history)

    # Step 2: Hybrid retrieval (dense + sparse)
    chunks = await retriever.retrieve(
        queries=expanded_queries,
        top_k=8,
        rerank=True
    )

    # Step 3: Context assembly with source attribution
    context = assemble_context(chunks, max_tokens=6000)

    # Step 4: Generation
    answer = await llm_generate(
        system=RAG_SYSTEM_PROMPT,
        context=context,
        query=state.query
    )

    return state.update(
        raw_answer=answer,
        source_chunks=chunks,
        execution_path="rag"
    )
```

### 4.3 SQL Agent Node

**Responsibility:** Translate natural language to SQL, execute it, and interpret results.

```python
async def sql_agent_node(state: AnalyticsState) -> AnalyticsState:
    # Step 1: Schema context retrieval (only relevant tables)
    schema_context = await schema_retriever.get_relevant_schema(state.query)

    # Step 2: Text-to-SQL generation
    sql_query = await generate_sql(
        query=state.query,
        schema=schema_context,
        dialect="snowflake"
    )

    # Step 3: SQL validation before execution
    validated_sql = validate_sql(sql_query)  # Syntax + injection check

    # Step 4: Execution with timeout
    results = await warehouse.execute(validated_sql, timeout_s=20)

    # Step 5: Result interpretation
    answer = await interpret_results(state.query, results)

    return state.update(
        raw_answer=answer,
        sql_query=sql_query,
        sql_results=results,
        execution_path="sql"
    )
```

### 4.4 Tool Agent Node

**Responsibility:** Select and call external tools (APIs, calculators, web search).

Uses a ReAct loop internally — the agent reasons, selects a tool, observes the result, and iterates until it has enough information.

```python
TOOLS = [
    web_search_tool,
    stock_price_tool,
    currency_converter_tool,
    calculator_tool,
    calendar_tool,
]

async def tool_agent_node(state: AnalyticsState) -> AnalyticsState:
    agent = create_react_agent(
        model=llm,
        tools=TOOLS,
        max_iterations=5,
        handle_parsing_errors=True
    )
    result = await agent.ainvoke({"input": state.query})
    return state.update(
        raw_answer=result["output"],
        tool_calls=result["intermediate_steps"],
        execution_path="tool"
    )
```

### 4.5 Verifier Node

**Responsibility:** Score the quality of the raw answer before it reaches the user.

```python
class VerificationResult(BaseModel):
    verdict: Literal["pass", "retry", "fail"]
    issues: list[str]
    confidence_score: float  # 0.0 – 1.0
    suggested_correction: str

async def verifier_node(state: AnalyticsState) -> AnalyticsState:
    checks = await asyncio.gather(
        check_relevance(state.query, state.raw_answer),
        check_hallucination(state.raw_answer, state.source_chunks),
        check_completeness(state.query, state.raw_answer),
        check_sql_result_consistency(state.sql_results, state.raw_answer),
    )
    verdict = aggregate_verdict(checks, state.retry_count)
    return state.update(verification=verdict)
```

**Verification checks:**

- **Relevance:** Does the answer address the query?
- **Grounding:** For RAG, is every claim traceable to a retrieved chunk?
- **Consistency:** For SQL, do the numbers in the answer match the raw query results?
- **Completeness:** Are all sub-queries from the router addressed?

### 4.6 Corrector Node

**Responsibility:** Diagnose failure and prepare a refined re-attempt.

```python
async def corrector_node(state: AnalyticsState) -> AnalyticsState:
    if state.retry_count >= MAX_RETRIES:
        return state.update(force_terminate=True)

    correction_plan = await diagnose_and_plan(
        query=state.query,
        failed_answer=state.raw_answer,
        issues=state.verification.issues,
        execution_path=state.execution_path
    )

    return state.update(
        retry_count=state.retry_count + 1,
        correction_instructions=correction_plan.instructions,
        strategy=correction_plan.new_strategy  # May switch from SQL to RAG
    )
```

### 4.7 Synthesizer Node

**Responsibility:** Format the final answer with citations, confidence, and structured metadata.

```python
async def synthesizer_node(state: AnalyticsState) -> AnalyticsState:
    final_response = AnalyticsResponse(
        answer=format_answer(state.raw_answer),
        citations=extract_citations(state.source_chunks),
        confidence=state.verification.confidence_score,
        execution_path=state.execution_path,
        sql_query=state.sql_query,          # Shown to power users
        latency_ms=state.elapsed_ms(),
        session_id=state.session_id,
    )
    return state.update(final_response=final_response)
```

---

## 5. Routing Logic

### Decision Matrix

| Query Signal                          | Strategy | Example                                      |
|---------------------------------------|----------|----------------------------------------------|
| Definition / explanation              | RAG      | "What is our churn calculation methodology?" |
| Aggregation / trend / comparison      | SQL      | "What was Q3 ARR by region?"                 |
| Real-time / external data             | Tool     | "What is USD/INR right now?"                 |
| Conversational / simple               | Direct   | "Thanks, can you summarize that?"            |
| Multi-hop (mixed signals)             | RAG→SQL  | "Explain CAC, then show me ours for 2024"    |

### Multi-Hop Decomposition

For queries that require multiple strategies, the router decomposes them into sub-queries, each routed independently, with results merged at the synthesizer.

```python
# Example decomposition
query = "Explain our revenue recognition policy and show total recognized revenue for H1 2024"

sub_queries = [
    SubQuery(text="Explain revenue recognition policy", strategy="rag"),
    SubQuery(text="Total recognized revenue H1 2024",  strategy="sql"),
]
```

---

## 6. RAG Pipeline

### Indexing Architecture

```
Raw Documents (PDF, Confluence, Notion, Sharepoint)
          │
          ▼
   Document Loader (LangChain)
          │
          ▼
   Chunking Strategy
   ├── Semantic chunking (preferred)     ← splits on meaning, not token count
   └── Recursive character splitting     ← fallback for unstructured text
          │
          ▼
   Embedding Model (text-embedding-3-large, 3072-dim)
          │
          ▼
   Dual Index in Weaviate
   ├── Dense vector index (HNSW)
   └── BM25 sparse index
```

### Retrieval Strategy: Hybrid + Reranking

```python
async def retrieve(query: str, top_k: int = 8) -> list[Chunk]:
    # Parallel dense + sparse retrieval
    dense_results, sparse_results = await asyncio.gather(
        weaviate_client.vector_search(query, k=20),
        weaviate_client.bm25_search(query, k=20),
    )

    # Reciprocal Rank Fusion
    fused = reciprocal_rank_fusion(dense_results, sparse_results)

    # Cross-encoder reranking
    reranked = await cross_encoder.rerank(query, fused[:20])

    return reranked[:top_k]
```

### Chunking Trade-offs

| Strategy           | Pros                              | Cons                               |
|--------------------|-----------------------------------|------------------------------------|
| Fixed token size   | Predictable, fast                 | Breaks semantic units              |
| Semantic chunking  | Preserves meaning                 | Variable size, slower indexing     |
| Document hierarchy | Preserves structure (headers)     | Complex to implement               |

**Decision:** Use semantic chunking with a 512-token soft cap and 10% overlap. Hierarchical metadata (doc title, section header) is stored alongside each chunk for context reconstruction.

---

## 7. Tool Execution Layer

### Tool Registry

```python
class ToolRegistry:
    tools: dict[str, BaseTool] = {}

    def register(self, tool: BaseTool, requires_auth: bool = False):
        self.tools[tool.name] = ToolWrapper(
            tool=tool,
            requires_auth=requires_auth,
            timeout_s=10,
            retry_policy=ExponentialBackoff(max_retries=3),
            rate_limiter=TokenBucketLimiter(rps=10)
        )
```

### Tool Safety Layer

Every tool call passes through:

1. **Input validation** — schema check before calling the tool
2. **Sanitization** — strip injection attempts from string inputs
3. **Timeout enforcement** — hard kill at 10s
4. **Output validation** — schema check on the response before passing to LLM
5. **Audit logging** — every call logged with user, tool, input, output, latency

### SQL-Specific Safety

```python
def validate_sql(query: str) -> str:
    # Allow only SELECT statements
    parsed = sqlglot.parse_one(query)
    if not isinstance(parsed, sqlglot.expressions.Select):
        raise SQLSecurityError("Only SELECT statements are permitted")

    # Block dangerous patterns
    blocked = ["DROP", "DELETE", "INSERT", "UPDATE", "EXEC", "xp_"]
    for keyword in blocked:
        if keyword.upper() in query.upper():
            raise SQLSecurityError(f"Blocked keyword: {keyword}")

    return query
```

---

## 8. Verification & Correction Loops

### Loop Architecture

```
Agent Output
     │
     ▼
┌────────────┐     score >= 0.8    ┌─────────────┐
│  Verifier  │ ─────────────────→  │ Synthesizer │ → User
└────────────┘                     └─────────────┘
     │
     │ score < 0.8 AND retry_count < MAX_RETRIES
     ▼
┌────────────┐
│ Corrector  │ → diagnose → adjust strategy → re-run agent
└────────────┘
     │
     │ retry_count >= MAX_RETRIES
     ▼
┌─────────────────────────┐
│ Graceful Degradation    │ → Return best attempt + uncertainty flag
└─────────────────────────┘
```

### Correction Strategies

| Failure Type             | Correction Action                                     |
|--------------------------|-------------------------------------------------------|
| Irrelevant answer        | Reformulate query, re-run same agent                  |
| Hallucination detected   | Switch to more grounded chunks, increase retrieval k  |
| SQL returns empty result | Relax filters, try alternate table, fall back to RAG  |
| Tool timeout             | Try alternate tool or fall back to LLM knowledge      |
| Incomplete answer        | Decompose into sub-queries, run in parallel           |

### MAX_RETRIES Policy

- Default: 2 retries (3 total attempts)
- Hard timeout across all retries: 25s
- On exhaustion: return best answer with `confidence: low` and `needs_review: true` flag

---

## 9. State Management

### AnalyticsState Schema

```python
from typing import Annotated
from langgraph.graph.message import add_messages

class AnalyticsState(TypedDict):
    # Input
    session_id:              str
    user_id:                 str
    query:                   str
    conversation_history:    Annotated[list[Message], add_messages]

    # Routing
    strategy:                Literal["rag", "sql", "tool", "direct"]
    sub_queries:             list[SubQuery]
    routing_confidence:      float

    # Execution
    raw_answer:              str
    source_chunks:           list[Chunk]
    sql_query:               Optional[str]
    sql_results:             Optional[list[dict]]
    tool_calls:              list[ToolCall]
    execution_path:          str

    # Verification
    verification:            VerificationResult
    retry_count:             int
    correction_instructions: Optional[str]
    force_terminate:         bool

    # Output
    final_response:          Optional[AnalyticsResponse]

    # Observability
    node_trace:              list[NodeTrace]
    start_time:              float
```

### Persistence: Redis Checkpointer

LangGraph's checkpointer is backed by Redis for:

- **Session resumption** — users can continue a conversation after disconnect
- **Retry safety** — if a node crashes, the graph resumes from the last checkpoint
- **Horizontal scaling** — any worker can pick up any session

```python
from langgraph.checkpoint.redis import RedisSaver

checkpointer = RedisSaver(
    redis_url=REDIS_URL,
    ttl_seconds=3600,         # Sessions expire after 1 hour of inactivity
    serializer=MsgPackSerializer()  # Faster than JSON for large states
)

graph = compiled_graph.with_config({"checkpointer": checkpointer})
```

### Conversation Memory

Long-term memory is handled separately from graph state:

```
Short-term  →  AnalyticsState.conversation_history  (in-graph, last 10 turns)
Long-term   →  PostgreSQL (full history, summarized per session)
Semantic    →  Vector store (past Q&A pairs, retrieved for similar future queries)
```

---

## 10. Observability Stack

### Tracing with LangSmith

Every graph execution emits a trace with:

- Node entry/exit timestamps
- LLM calls with token counts and model name
- Tool calls with input/output
- State diffs at each node

```python
from langchain_core.tracers import LangChainTracer

tracer = LangChainTracer(project_name="analytics-assistant-prod")

async def invoke_with_tracing(query: str, session_id: str):
    config = {
        "callbacks": [tracer],
        "configurable": {"thread_id": session_id},
        "tags": ["prod", f"user:{user_id}"]
    }
    return await graph.ainvoke({"query": query}, config=config)
```

### Metrics with Prometheus

```python
# Key metrics exposed at /metrics
query_latency_histogram = Histogram(
    "agent_query_latency_seconds",
    "End-to-end query latency",
    labelnames=["strategy", "verdict"]
)

node_latency_histogram = Histogram(
    "agent_node_latency_seconds",
    "Per-node latency",
    labelnames=["node_name"]
)

retry_counter = Counter(
    "agent_retry_total",
    "Number of verification retries",
    labelnames=["strategy", "failure_reason"]
)

token_usage_counter = Counter(
    "agent_token_usage_total",
    "LLM token consumption",
    labelnames=["model", "node_name", "token_type"]
)

verification_score_histogram = Histogram(
    "agent_verification_score",
    "Distribution of verifier confidence scores",
    labelnames=["strategy"]
)
```

### Grafana Dashboard Panels

| Panel                         | Alert Threshold              |
|-------------------------------|------------------------------|
| p99 end-to-end latency        | > 30s                        |
| Retry rate                    | > 20% of requests            |
| Verification pass rate        | < 80%                        |
| SQL execution errors          | > 5% of SQL queries          |
| LLM error rate                | > 1%                         |
| Token spend per hour          | > budget threshold           |

### Distributed Tracing with Jaeger

For multi-service request flows (API Gateway → LangGraph Workers → Vector Store → Warehouse), OpenTelemetry spans are emitted and collected in Jaeger. Each span includes:

- `session_id` and `user_id` as baggage
- Node name as the span operation name
- External service calls as child spans

---

## 11. Scalability & Deployment

### Worker Architecture

```
                    ┌──────────────────────────┐
                    │     Load Balancer         │
                    │     (AWS ALB / Nginx)     │
                    └────────────┬─────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
   ┌──────────▼──────┐  ┌────────▼────────┐  ┌─────▼───────────┐
   │  FastAPI Worker │  │  FastAPI Worker  │  │  FastAPI Worker  │
   │  (LangGraph)    │  │  (LangGraph)     │  │  (LangGraph)     │
   └──────────┬──────┘  └────────┬─────────┘  └─────┬───────────┘
              │                  │                   │
              └──────────────────┼───────────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │    Redis Cluster          │
                    │  (State / Checkpointing)  │
                    └──────────────────────────┘
```

### Kubernetes Deployment

```yaml
# langgraph-worker deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: analytics-agent-worker
spec:
  replicas: 4
  selector:
    matchLabels:
      app: analytics-agent
  template:
    spec:
      containers:
      - name: worker
        image: analytics-agent:latest
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
        env:
        - name: MAX_CONCURRENT_GRAPHS
          value: "50"           # Per worker
        - name: LLM_TIMEOUT_S
          value: "25"
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: url
```

### Async Execution Model

All LangGraph nodes are `async def`. The FastAPI endpoint uses `asyncio` throughout:

```python
@app.post("/query")
async def handle_query(request: QueryRequest) -> QueryResponse:
    async with semaphore:  # Bound concurrency per worker
        result = await graph.ainvoke(
            input={"query": request.query, "session_id": request.session_id},
            config={"configurable": {"thread_id": request.session_id}}
        )
    return QueryResponse(**result["final_response"])
```

### Caching Strategy

| Cache Layer         | What is Cached                    | TTL       | Store   |
|---------------------|-----------------------------------|-----------|---------|
| Embedding cache     | Query → embedding vector          | 24 hours  | Redis   |
| SQL result cache    | Identical SQL query → results     | 15 min    | Redis   |
| Schema cache        | Table schema metadata             | 1 hour    | Redis   |
| LLM response cache  | Exact query + context → response  | 1 hour    | Redis   |

Semantic caching (using embedding similarity for LLM responses) reduces LLM calls by ~30% in practice for repetitive enterprise query patterns.

---

## 12. Tech Stack Summary

### Core Framework

| Component            | Choice                        | Rationale                                                    |
|----------------------|-------------------------------|--------------------------------------------------------------|
| Agent orchestration  | LangGraph 0.2+                | Native support for cycles, checkpointing, conditional edges  |
| API framework        | FastAPI                       | Async-native, Pydantic validation, OpenAPI docs              |
| LLM (routing/verify) | GPT-4o-mini / Claude Haiku    | Fast, cheap for classification tasks                         |
| LLM (generation)     | GPT-4o / Claude Sonnet        | High quality for answer synthesis                            |
| Embeddings           | text-embedding-3-large        | Best-in-class retrieval quality at reasonable cost           |

### Data & Storage

| Component        | Choice          | Rationale                                         |
|------------------|-----------------|---------------------------------------------------|
| Vector store     | Weaviate        | Hybrid (dense + BM25), multi-tenancy, production-ready |
| Data warehouse   | Snowflake       | Enterprise standard, Python connector, row-level security |
| State store      | Redis Cluster   | Low-latency checkpointing, pub/sub for streaming  |
| Session history  | PostgreSQL      | Reliable relational store for conversation logs   |

### Observability

| Component           | Choice        | Rationale                                          |
|---------------------|---------------|----------------------------------------------------|
| LLM tracing         | LangSmith     | Native LangGraph integration, token tracking       |
| Metrics             | Prometheus    | Industry standard, rich ecosystem                  |
| Dashboards          | Grafana       | Pairs naturally with Prometheus                    |
| Distributed tracing | Jaeger        | OpenTelemetry-compatible, free                     |
| Log aggregation     | ELK Stack     | Full-text search on structured logs                |

### Infrastructure

| Component      | Choice             | Rationale                                              |
|----------------|--------------------|--------------------------------------------------------|
| Deployment     | Kubernetes (EKS)   | HPA for worker scaling, pod isolation per service      |
| Container      | Docker             | Reproducible builds, layer caching for dependencies    |
| CI/CD          | GitHub Actions     | Native integration, matrix builds for testing          |
| Secrets        | AWS Secrets Manager| Rotation support, fine-grained IAM                     |
| Load balancer  | AWS ALB            | WebSocket support, sticky sessions optional            |

---

## 13. Trade-off Analysis

### Trade-off 1: Single Graph vs. Multi-Graph Architecture

**Single graph (chosen):** All agents live in one LangGraph graph. State flows through a single object.

- ✅ Simpler deployment — one service to manage
- ✅ State sharing is trivial — no serialization between services
- ✅ Easier to debug — one trace per query
- ❌ Harder to scale individual agents independently
- ❌ A bug in one node can affect all query types

**Multi-graph (alternative):** Each agent is a separate compiled graph, orchestrated by a supervisor graph.

- ✅ Independent scaling and deployment per agent
- ✅ Fault isolation
- ❌ Inter-graph communication requires serialization
- ❌ Significantly more complex to trace and debug

**Decision:** Start with a single graph. Migrate high-load nodes (SQL agent) to independent graphs when p99 latency exceeds SLO.

---

### Trade-off 2: Synchronous vs. Streaming Response

**Synchronous (chosen for v1):** Wait for the full answer, then return.

- ✅ Simpler client integration
- ✅ Verification can happen before any text is shown to the user
- ❌ User sees nothing for 5–10s on complex queries

**Streaming (v2 target):** Stream tokens as they are generated, with a verification pass happening asynchronously.

- ✅ Perceived latency drops dramatically
- ❌ Cannot verify before showing — must post-correct or show disclaimer
- ❌ WebSocket management adds infrastructure complexity

**Decision:** Ship synchronous v1. Add streaming in v2 with an "Unverified — generating" indicator in the UI.

---

### Trade-off 3: Reranking — Always vs. Selective

**Always rerank:** Every RAG retrieval passes through a cross-encoder.

- ✅ Consistently higher retrieval quality
- ❌ Adds 400–800ms per query (cross-encoders are slow)

**Selective reranking (chosen):** Rerank only when routing confidence is low or when the query is long-tail.

- ✅ Keeps p50 latency low for common queries
- ❌ Slightly inconsistent quality depending on routing confidence score

---

### Trade-off 4: Verifier Model Size

**Large model verifier (GPT-4o):** More accurate hallucination detection.

- ✅ Catches subtle inconsistencies
- ❌ Doubles LLM cost per query
- ❌ Adds 1–2s to every response

**Small model verifier (chosen, GPT-4o-mini):** Faster and cheaper.

- ✅ < 400ms verification overhead
- ❌ May miss nuanced hallucinations
- ❌ Requires very explicit verification prompts

**Mitigation:** Use rule-based checks (number consistency, source grounding) as a primary filter, reserving LLM-based verification for ambiguous cases.

---

### Trade-off 5: SQL Agent — Chain vs. ReAct

**Chain (single-shot):** Generate SQL once, execute, interpret.

- ✅ Fast — one LLM call
- ❌ High failure rate on complex joins or ambiguous schemas

**ReAct loop (chosen):** Generate SQL → inspect schema if needed → refine → execute.

- ✅ Higher success rate on complex queries
- ❌ 2–3 LLM calls in the worst case
- ❌ Harder to bound latency

---

## 14. Failure Modes & Mitigations

| Failure Mode                    | Impact                      | Mitigation                                                  |
|---------------------------------|-----------------------------|-------------------------------------------------------------|
| LLM API timeout                 | Query fails entirely        | 25s hard timeout, fallback to cached similar answer         |
| Vector store unavailable        | RAG queries fail            | Circuit breaker, degrade to direct LLM with disclaimer      |
| SQL warehouse timeout           | SQL queries fail            | 20s query timeout, query optimization hints in prompt       |
| Redis checkpointer down         | Session state lost          | In-memory fallback checkpointer, stateless retry            |
| Infinite correction loop        | Query hangs indefinitely    | MAX_RETRIES=2, hard wall-clock timeout of 25s               |
| Prompt injection via user input | Malicious SQL / tool misuse | Input sanitization, SQL allow-list (SELECT only), sandboxed tool execution |
| LLM cost spike                  | Budget overrun              | Token budget per query (8k max), alert at 80% of hourly budget |
| Hallucinated SQL column names   | Query execution error       | Schema validation layer before execution, error → corrector |
| Stale embeddings after doc update | Incorrect RAG answers     | Incremental re-indexing pipeline triggered on document change |

---

## Appendix: Key Design Principles

**1. Fail gracefully, always.** Every failure path returns *something* to the user — even if it's "I couldn't find a confident answer, here's what I attempted." Silent failures are worse than honest uncertainty.

**2. Make every decision observable.** Routing decisions, verification scores, retry counts, and token usage are first-class metrics — not afterthoughts. You cannot improve what you cannot measure.

**3. Separate fast paths from slow paths.** Direct LLM responses should never be held up by the same infrastructure as complex SQL queries. The router node's job is partly to protect the fast path.

**4. State is the contract.** The `AnalyticsState` TypedDict is the API between every node. If a node's output isn't in the state schema, it doesn't exist to downstream nodes. This discipline prevents hidden coupling.

**5. Agents are stateless, graphs are stateful.** Each node function is a pure(ish) function: given state in, return state out. The graph manages all persistence. This makes individual nodes trivially testable.

---

*Design version 1.0 — March 2026*
