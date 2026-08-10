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

---
--
--
# System Design: Enterprise Analytics Assistant (LangGraph Multi-Agent)

**Status:** Draft v1
**Owner:** Data Science / AI Platform
**Scope:** Query routing across RAG, tool execution (SQL/API), and direct LLM response, with verification/correction loops, observability, and production scalability.

---

## 1. Problem Statement & Goals

**You are designing a production-grade multi-agent system using LangGraph for an enterprise
analytics assistant.
The system should:
Handle user queries
Decide whether to use RAG, tool execution (SQL / APIs), or direct LLM response
Support verification and correction loops
Be observable and scalable in production**

Build a multi-agent orchestration layer that lets enterprise users ask natural-language analytics questions and get correct, grounded, auditable answers, regardless of whether the answer requires:

- Retrieval from internal knowledge bases (policy docs, runbooks, metric definitions)
- Structured data access (SQL over a warehouse, internal APIs)
- Pure reasoning/summarization from the LLM itself

### Non-negotiable requirements

| Requirement | Detail |
|---|---|
| Correctness | Answers must be verified/grounded before returning to the user |
| Auditability | Every decision (route chosen, SQL run, sources cited) must be traceable |
| Latency | P50 < 4s for direct/RAG paths, P50 < 8s for SQL/tool paths |
| Scalability | Support concurrent multi-tenant load, horizontally scalable workers |
| Safety | No unguarded SQL execution, no PII leakage, no hallucinated citations |
| Extensibility | New tools/data sources addable without redesigning the graph |

---

## 2. High-Level Architecture

```
                                 ┌─────────────────────────┐
                                 │      API Gateway /       │
                                 │   Auth (OIDC/mTLS)       │
                                 └────────────┬─────────────┘
                                              │
                                 ┌────────────▼─────────────┐
                                 │   Orchestration Service   │
                                 │  (FastAPI + LangGraph     │
                                 │   Runtime, stateless)     │
                                 └────────────┬─────────────┘
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │                         │                         │
             ┌──────▼──────┐          ┌───────▼───────┐         ┌───────▼───────┐
             │  Checkpoint  │          │   LangGraph    │         │   Trace/Log   │
             │  Store       │◄────────►│   Graph Exec   │────────►│   Sink        │
             │ (Postgres/   │          │   (per-thread) │         │ (OTel + LLM   │
             │  Redis)      │          └───────┬───────┘         │  observability)│
             └──────────────┘                  │                 └───────────────┘
                                                │
              ┌────────────────┬────────────────┼────────────────┬────────────────┐
              │                │                │                │                │
       ┌──────▼─────┐   ┌──────▼─────┐   ┌──────▼─────┐   ┌──────▼─────┐   ┌──────▼─────┐
       │  Router /  │   │    RAG     │   │    Tool     │   │  Direct     │   │ Verifier / │
       │  Planner   │   │   Node     │   │  Execution  │   │  LLM Node   │   │ Corrector  │
       │   Node     │   │            │   │    Node     │   │             │   │   Node     │
       └────────────┘   └─────┬──────┘   └─────┬──────┘   └─────────────┘   └─────┬──────┘
                               │                │                                  │
                        ┌──────▼──────┐  ┌──────▼───────┐                  ┌───────▼───────┐
                        │ Vector Store │  │ SQL Engine /  │                  │ Response      │
                        │ (pgvector /  │  │ Warehouse +   │                  │ Synthesizer   │
                        │  OpenSearch) │  │ API Gateway   │                  │ Node          │
                        └──────────────┘  └───────────────┘                  └───────────────┘
```

---

## 3. LangGraph State Design

LangGraph models the whole interaction as a typed state object that flows through nodes. Keep it flat, serializable, and checkpoint-friendly.

```python
from typing import TypedDict, Literal, Optional
from langgraph.graph import add_messages
from typing_extensions import Annotated

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]      # conversation history
    user_query: str
    route: Optional[Literal["rag", "sql", "api", "direct", "clarify"]]
    route_confidence: float
    retrieved_context: Optional[list[dict]]       # docs + metadata + scores
    tool_calls: Optional[list[dict]]               # SQL/API calls made, params, results
    draft_answer: Optional[str]
    verification: Optional[dict]                   # {"grounded": bool, "issues": [...], "score": float}
    correction_attempts: int
    final_answer: Optional[str]
    citations: Optional[list[dict]]
    trace_id: str
    tenant_id: str
    user_id: str
    audit_log: list[dict]
```

Design choices:

- **`audit_log`** is appended to (never overwritten) at every node — this is what feeds the compliance/audit trail, independent of the tracing backend.
- **`correction_attempts`** is a bounded counter used for loop control (see §6).
- State is kept small and JSON-serializable so it can be checkpointed cheaply and inspected by humans during incident review.

---

## 4. Graph Topology

```
        ┌───────────┐
        │  START    │
        └─────┬─────┘
              │
        ┌─────▼──────┐
        │  Router /  │──── low confidence ───►┌────────────┐
        │  Planner   │                        │  Clarify   │──► END (ask user)
        └─────┬──────┘                        └────────────┘
              │
   ┌──────────┼───────────┬────────────┐
   │          │           │            │
┌──▼───┐  ┌───▼───┐  ┌────▼────┐  ┌────▼────┐
│ RAG  │  │  SQL   │  │  API    │  │ Direct  │
│ Node │  │ Node   │  │  Node   │  │  LLM    │
└──┬───┘  └───┬────┘  └────┬────┘  └────┬────┘
   │          │            │            │
   └──────────┴─────┬──────┴────────────┘
                     │
              ┌──────▼───────┐
              │  Verifier /   │
              │  Grounding    │
              │  Check Node   │
              └──────┬───────┘
                      │
          ┌───────────┴────────────┐
     fail │                        │ pass
          ▼                        ▼
   ┌──────────────┐        ┌───────────────┐
   │  Corrector /  │        │   Response     │
   │  Re-planner   │        │   Synthesizer  │
   │  (loops back  │        └───────┬───────┘
   │  to Router)   │                │
   └──────┬────────┘                ▼
          │                       END
   attempts < N ──► Router
   attempts >= N ──► "answer with caveats" ──► END
```

### 4.1 Router / Planner Node

- Classifies intent using a small, fast model (or a fine-tuned classifier) plus rule-based overrides (e.g., regex for "as of <date>", "compare", "trend" → likely SQL; "policy", "how do I", "what does X mean" → likely RAG).
- Outputs `route` + `route_confidence`. Below a confidence threshold, route to `clarify` instead of guessing — this is cheaper than a wrong tool call and reduces downstream correction-loop load.
- Also responsible for **decomposition**: a query needing both RAG (definition of a metric) and SQL (its value) is split into subtasks and routed as a short plan, not forced into one branch. This can be implemented as a `Send()`-based fan-out in LangGraph to run RAG and SQL nodes in parallel, then merge in the synthesizer.

### 4.2 RAG Node

- Hybrid retrieval (dense + keyword/BM25) over the vector store, with metadata filters for tenant isolation.
- Reranking pass (cross-encoder or LLM-based) before context is passed downstream.
- Every retrieved chunk carries `{doc_id, source, score, chunk_text}` — this becomes the citation list, not a post-hoc guess.

### 4.3 Tool Execution Node (SQL / API)

This is the highest-risk node and gets the most guardrails:

- **NL→SQL**: LLM generates SQL against a curated schema/semantic layer (not the raw production schema) — reduces hallucinated joins and exposes only sanctioned tables/views.
- **Static validation before execution**: parse the SQL (e.g., via `sqlglot`), reject anything outside an allowlist of statement types (`SELECT` only), enforce row limits, enforce mandatory `WHERE` filters for tenant/PII scoping.
- **Execution** happens through a policy-gated data-access service, not directly against the warehouse from the agent process — this is the enforcement point for row/column-level security.
- **API tools**: registered via a typed tool registry (name, JSON schema for args, auth scope, timeout, retry policy). The LLM only ever sees the schema, never raw credentials.
- Results (SQL rows, API payloads) are summarized/truncated before being placed back into state to control token growth.

### 4.4 Direct LLM Node

- For queries that need no external grounding (rephrasing, summarizing something already in the conversation, general reasoning). Still passes through the verifier — direct answers are exactly where a model is most tempted to state things confidently and wrongly.

### 4.5 Verifier / Grounding Check Node

- Checks the draft answer against the actual evidence in state (`retrieved_context` and/or `tool_calls` results), not against the model's own confidence:
  - **Groundedness**: every factual claim traceable to a retrieved chunk or tool result (NLI-style entailment check or an LLM-as-judge prompt constrained to compare claim vs. source).
  - **Numeric consistency**: for SQL-derived answers, re-derive key numbers programmatically and diff against what the draft states, rather than trusting the LLM's arithmetic.
  - **Policy/PII check**: no leakage of restricted fields.
- Emits a structured `verification` object with a pass/fail and a list of specific issues (not just a score) — this is what the corrector needs to act on.

### 4.6 Corrector / Re-planner Node

- On failure, decides *why* it failed and routes accordingly:
  - Missing/insufficient evidence → back to Router with a refined sub-query (e.g., expand retrieval, adjust SQL filter).
  - Wrong route chosen → re-route entirely (e.g., RAG was tried but the question actually needed SQL).
  - Tool error (timeout, bad SQL) → regenerate the tool call with the error message included as context.
- Bounded by `correction_attempts` (recommend max 2–3). On exhaustion, the response synthesizer returns a best-effort answer **with explicit caveats and reduced confidence**, never a silently degraded "confident" answer.

### 4.7 Response Synthesizer Node

- Merges parallel branches (if decomposition occurred), attaches citations, formats the final answer, and writes the final audit record.

---

## 5. Cyclic Control Flow (Why LangGraph, Specifically)

LangGraph is the right fit here (over a plain DAG orchestrator) because:

- The **verify → correct → re-route** loop is inherently cyclic, not a linear DAG — LangGraph's conditional edges handle this natively.
- **Checkpointing** (`Checkpointer` / `Postgres`/`Redis` backends) gives durable, resumable execution per `thread_id` — critical for long-running tool calls and for human-in-the-loop interrupts.
- **`interrupt()`** support enables human approval gates (e.g., "this SQL will scan 200M rows, approve?" or a low-confidence answer routed to a human reviewer) without re-architecting the graph.
- Native support for parallel fan-out/fan-in (`Send`) covers the decomposition case in §4.1 cleanly.

---

## 6. Loop & Cost Control

Uncontrolled agent loops are the single biggest production risk. Controls:

| Control | Mechanism |
|---|---|
| Max correction attempts | Hard cap in state (`correction_attempts`), enforced in the conditional edge, not just prompted |
| Max total node visits | Global step counter in state; graph aborts to a safe fallback past a ceiling |
| Per-node timeouts | Wrapped at the node level; tool nodes additionally have per-call timeouts |
| Token/cost budget per request | Running token counter in state; router uses cheaper model, deep reasoning reserved for verifier/corrector |
| Circuit breaker on tool/data source | If a downstream API/warehouse is failing repeatedly, short-circuit to "service degraded" response instead of retrying into the wall |

---

## 7. Observability

Three layers, all keyed by a single `trace_id` propagated through state:

1. **Execution tracing** — LangSmith (or OpenTelemetry + a compatible backend) instrumented at every node boundary: inputs, outputs, latency, token usage, model/version used. This is what lets an engineer replay exactly what happened for a given `trace_id`.
2. **Business/audit logging** — the `audit_log` field in state, persisted to an append-only store (e.g., a dedicated Postgres table or object storage), capturing: route chosen and why, SQL executed (verbatim) and against which dataset, sources cited, verification outcome, correction attempts, final answer. This is the artifact that satisfies compliance/audit requirements independent of any third-party tracing vendor.
3. **Metrics** — exported via Prometheus/OTel metrics:
   - Route distribution (RAG vs SQL vs API vs direct vs clarify)
   - Verification pass rate on first attempt vs after correction
   - Correction-loop trigger rate and average attempts
   - P50/P95/P99 latency per node and end-to-end
   - Tool error rate, SQL rejection rate (by the static validator)
   - Token cost per request, per route

Alerting thresholds should be set on: verification failure rate spike, correction-loop exhaustion rate, tool error rate, and latency P95 breaches — these are the leading indicators of a degrading system, well before users complain.

---

## 8. Scalability & Deployment

- **Stateless orchestration workers**: the FastAPI/LangGraph process itself holds no session state; all state lives in the checkpoint store, so workers scale horizontally behind a load balancer with no sticky sessions required.
- **Checkpoint store**: Postgres (durable, queryable for audit) or Redis (lower latency) depending on retention needs — Postgres is the safer default for an enterprise/regulated context since it doubles as an audit source.
- **Async execution for long tool calls**: SQL/API nodes should be async; for genuinely long-running queries, use LangGraph's interrupt/resume so the HTTP layer isn't held open — poll or push (webhook/SSE) for completion instead.
- **Model tiering**: cheap/fast model for routing and simple direct answers; stronger model reserved for SQL generation, verification, and correction — this is where most of the cost and latency budget should go, not on classification.
- **Caching**:
  - Semantic cache on RAG queries (embedding similarity) to avoid redundant retrieval.
  - Result cache on SQL node keyed by normalized query + filters, with a short TTL appropriate to data freshness requirements.
- **Multi-tenancy**: `tenant_id` threaded through state and enforced at the data-access layer (row-level security in the warehouse, filtered vector namespaces), not just trusted from the prompt.
- **Horizontal scale-out**: run the orchestration service as a Kubernetes deployment with HPA on queue depth/CPU; tool-execution and RAG retrieval can be separate services scaled independently from the orchestration layer if load profiles diverge.

---

## 9. Security & Governance

- **AuthN/AuthZ** at the API gateway (OIDC), with per-tenant/per-role scopes passed into state and enforced again at the data-access layer (defense in depth — never trust the LLM's tool call as the authorization boundary).
- **No direct DB credentials in the agent process** — all SQL execution proxied through a policy-gated data service.
- **PII handling**: redaction/masking at retrieval and at tool-result ingestion, before those results ever reach an LLM context window.
- **Prompt-injection containment**: retrieved documents and tool outputs are treated as data, never as instructions — enforce this with a system prompt that explicitly demotes retrieved/tool content to "untrusted context," plus a lightweight injection classifier on retrieved content for high-sensitivity corpora.
- **Full replayability**: given a `trace_id`, an auditor should be able to reconstruct the entire decision path — route, evidence, SQL text, verification outcome — without needing the original LLM to "explain itself" after the fact.

---

## 10. Failure Modes & Fallbacks

| Failure | Fallback |
|---|---|
| Vector store unavailable | Route to SQL/API if plausible, else direct LLM with explicit "no internal sources available" caveat |
| Warehouse/API timeout | Retry with backoff (bounded), then circuit-break to a cached/last-known-good result if available, else fail gracefully with explanation |
| Verifier itself errors | Fail closed — return answer with an "unverified" flag rather than blocking indefinitely |
| Correction loop exhausted | Return best-effort answer with explicit confidence caveat and offer human escalation |
| Router misclassifies repeatedly (seen via metrics) | Feed logged misroutes back into router fine-tuning/prompt refinement — this is a continuous-improvement loop, not a one-time launch task |

---

## 11. Suggested Tech Stack

| Layer | Choice |
|---|---|
| Orchestration | LangGraph (Python), FastAPI service wrapper |
| Checkpointing | Postgres (`PostgresSaver`) |
| Vector store | pgvector or OpenSearch (reuse existing enterprise search infra if present) |
| SQL semantic layer | dbt/LookML-style curated views, not raw prod schema |
| Tracing | LangSmith or OpenTelemetry → Grafana/Jaeger |
| Metrics | Prometheus + Grafana |
| Audit store | Append-only Postgres table / object storage with WORM policy |
| Deployment | Kubernetes, HPA on queue depth |
| Model routing | Small model for classification/routing, larger model for generation/verification |

---

## 12. Open Design Questions

- Human-in-the-loop threshold: at what verification-confidence cutoff does a response route to a human reviewer instead of auto-answering?
- Data freshness vs. cache TTL trade-off for the SQL result cache in fast-moving metrics.
- Whether decomposition (parallel RAG+SQL fan-out) should be router-driven or handled by a separate lightweight planning pass to keep the router node fast.
