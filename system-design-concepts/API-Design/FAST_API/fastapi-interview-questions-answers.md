# FastAPI Scenario-Based Interview Questions — With Answers

---

## System Design & Architecture

### 1. High-traffic endpoint (500 req/sec ML model)
**High-traffic endpoint: "You have a /predict endpoint serving an ML model that gets 500 requests/sec. How would you design it in FastAPI to avoid blocking the event loop? Would you use async def or def, and why?"**


Use `async def` only if the model inference and all I/O inside the handler are truly non-blocking (async DB driver, async HTTP client for feature lookups, etc.). If the model call itself is a blocking synchronous call (most `sklearn`/`xgboost`/`torch` inference is), then:

- Either keep the endpoint as `def` (FastAPI runs sync `def` endpoints in a threadpool automatically via Starlette), or
- Use `async def` but offload the blocking inference to `run_in_threadpool` / `asyncio.to_thread`.

Also add: connection pooling for any DB/feature store, response caching where safe, and horizontal scaling (multiple uvicorn/gunicorn workers) since a single Python process is still GIL-bound for CPU work.

**Key point interviewers look for**: understanding that `async def` doesn't make CPU-bound code faster — it only helps when you're waiting on I/O.

---

### 2. Multiple models, one service
*Multiple models, one service: "Your team owns 5 ML models (fraud, AML, credit risk, etc.) that need to be served via one API gateway. How would you structure the FastAPI project — routers, versioning, shared dependencies?"*

Structure:
```
app/
  main.py
  routers/
    fraud.py
    aml.py
    credit_risk.py
  core/
    config.py
    dependencies.py   # shared auth, db session, logging
  models/
    schemas/           # pydantic request/response models per domain
  services/
    fraud_service.py   # business logic, calls the actual model
  ml/
    fraud_model.py      # model loading/inference wrapper
```
- Use `APIRouter` per domain, mounted with a version prefix: `app.include_router(fraud.router, prefix="/v1/fraud")`.
- Shared dependencies (auth, tenant context, rate limiting, logging) live in `core/dependencies.py` and are injected via `Depends()`.
- Each model is loaded once at startup (via FastAPI's `lifespan` context manager) and stored in `app.state`, not reloaded per request.
- Version at the router/prefix level (`/v1`, `/v2`) so models can evolve independently.

---

### 3. Sync legacy DB driver inside async endpoint
**Sync legacy code: "You need to call a legacy synchronous database driver (no async support) inside an async endpoint. What happens if you just await it directly? How would you actually integrate it?"**

If you `await` a sync call directly, it doesn't work — sync functions aren't awaitable, and even if you wrap them incorrectly, calling a blocking driver inside `async def` blocks the entire event loop, stalling every other concurrent request.

Correct approach:
```python
from starlette.concurrency import run_in_threadpool

@app.get("/legacy")
async def legacy_endpoint():
    result = await run_in_threadpool(legacy_sync_call, arg1, arg2)
    return result
```
Alternatively, just declare the endpoint as `def` (not `async def`) — FastAPI will automatically run it in a threadpool. Long-term, migrate to an async driver (e.g., `asyncpg` instead of `psycopg2`) if the DB supports it.

---

### 4. Long-running task (2-minute report generation)
**Long-running task: "A user hits an endpoint that triggers a report generation taking 2 minutes. How do you design this so the API doesn't time out — walk through the request/response flow?"**

Don't block the request-response cycle. Pattern:

1. Client `POST /reports` → server validates request, creates a job record with `status=pending`, enqueues work (Celery/RQ/arq, or FastAPI `BackgroundTasks` for lighter cases), returns `202 Accepted` with a `job_id` and a `Location`/polling URL immediately.
2. Worker processes the job asynchronously, updates job status in DB/Redis.
3. Client polls `GET /reports/{job_id}` for status, or you push updates via WebSocket/SSE, or notify via webhook/email when done.

This avoids client-side timeouts, load balancer timeouts (typically 30–60s), and worker thread starvation.

---

## Concurrency & Performance

### 5. CPU-bound feature engineering hurting async
**CPU-bound work: "Your endpoint does CPU-heavy feature engineering before calling a model. Why can async def hurt you here, and how would you fix it (thread pool, process pool, background worker)?"**

If `async def` runs CPU-heavy pandas/numpy transforms synchronously inside it, that code still blocks the single event loop thread — no other request can be handled during that time, even though the endpoint "looks" async.

Fixes:
- **Thread pool**: `await run_in_threadpool(feature_engineering_fn, data)` — helps if the CPU work releases the GIL periodically (numpy/pandas often do for vectorized ops).
- **Process pool**: `concurrent.futures.ProcessPoolExecutor` via `loop.run_in_executor` — better for pure-Python CPU-bound code that holds the GIL, since it uses separate processes.
- **Background worker/task queue**: for very heavy work, offload to Celery/RQ entirely and return a job ID (see Q4).

---

### 6. Connection pooling under concurrency
**Connection pooling: "How would you manage a database connection pool across thousands of concurrent async requests without exhausting connections?"**

- Use an async connection pool matched to your driver (`asyncpg` pool, SQLAlchemy async engine with `pool_size`/`max_overflow`, or `databases` library).
- Set pool size based on `(number of workers) × (pool_size per worker) ≤ DB max_connections`.
- Use a single pool per process, created in the `lifespan` startup hook, shared across requests via `app.state` or dependency injection — never create a new connection per request.
- Set sensible `pool_timeout` so requests fail fast (with a clear error) rather than queuing indefinitely when the pool is exhausted.
- Monitor pool saturation metrics; add read replicas or a pgbouncer-style external pooler if a single app-level pool isn't enough.

---

### 7. Preventing timeout cascades from a slow downstream call'\
**Timeout cascades: "Your FastAPI service calls three downstream services in sequence. One is slow. How do you prevent that from cascading into your entire service timing out, and how would you test for this?"**

- Set explicit **per-call timeouts** on every downstream HTTP call (e.g., `httpx.AsyncClient(timeout=2.0)`), not just a global timeout.
- Call independent downstream services **concurrently** with `asyncio.gather()` instead of sequentially, where the calls don't depend on each other.
- Add a **circuit breaker** (e.g., via `pybreaker` or custom logic) so a consistently slow/failing service gets short-circuited instead of retried every time.
- Define a fallback/default response when a non-critical downstream call fails or times out, rather than failing the whole request.
- Testing: simulate a slow dependency with a mock that sleeps beyond the timeout, and assert the endpoint still returns within its own SLA (e.g., using `pytest-asyncio` + `respx`/`httpx` mocking, or `asyncio.wait_for` in tests).

---

## Validation & Data Modeling

### 8. Conditional/nested validation for fraud-alert payloads

**Complex nested payloads: "Design Pydantic models for a fraud-alert payload that includes nested transaction details, customer metadata, and a list of risk signals — with different required fields depending on transaction type. How do you handle that conditional validation?"**

Use nested Pydantic models plus a **validator** (or `model_validator` in Pydantic v2) for conditional logic:

```python
from pydantic import BaseModel, model_validator
from typing import Literal, Optional
from enum import Enum

class TransactionType(str, Enum):
    wire = "wire"
    card = "card"

class RiskSignal(BaseModel):
    signal_type: str
    score: float

class Customer(BaseModel):
    customer_id: str
    kyc_level: str

class Transaction(BaseModel):
    txn_id: str
    txn_type: TransactionType
    amount: float
    beneficiary_country: Optional[str] = None
    card_last4: Optional[str] = None

    @model_validator(mode="after")
    def check_conditional_fields(self):
        if self.txn_type == TransactionType.wire and not self.beneficiary_country:
            raise ValueError("beneficiary_country required for wire transactions")
        if self.txn_type == TransactionType.card and not self.card_last4:
            raise ValueError("card_last4 required for card transactions")
        return self

class FraudAlert(BaseModel):
    customer: Customer
    transaction: Transaction
    risk_signals: list[RiskSignal]
```

This gives clear, structured 422 errors instead of silently accepting malformed data.

---

### 9. Adding a required field without breaking existing clients

**Backward compatibility: "You need to add a new required field to a request model without breaking existing clients. How do you version the API and manage the schema evolution?"**

- Never make the new field required on an existing version — either give it a default value, or introduce a new API version (`/v2/...`) where it's required.
- Use API versioning at the URL or header level (`Accept: application/vnd.myapi.v2+json`), and keep the `/v1` router serving the old schema until clients migrate (deprecation window with a sunset date, communicated via response headers like `Deprecation` / `Sunset`).
- Pydantic makes this easy: define `TransactionV1` and `TransactionV2` as separate models, or make the field `Optional[str] = None` with backward-compatible defaulting logic in the service layer.
- Track schema changes with contract tests (e.g., against an OpenAPI spec diff) so breaking changes are caught in CI.

---

### 10. Partial validation failures in a 1,000-record batch
**Partial validation failures: "A batch endpoint accepts 1,000 records; 12 fail validation. Do you reject the whole batch or process the valid ones? How would you design the response schema to communicate partial success?"**

Generally: **process valid records, report failures explicitly** — don't reject the whole batch unless the domain requires atomicity (e.g., a financial ledger transaction where partial application is dangerous).

Response design:
```json
{
  "accepted": 988,
  "rejected": 12,
  "results": [
    {"index": 4, "status": "rejected", "errors": ["amount must be positive"]},
    {"index": 57, "status": "accepted", "record_id": "..."}
  ]
}
```
Implementation: accept a `list[dict]` (not a strict `list[Transaction]`) at the top level, validate each record individually in a loop with `Transaction.model_validate(record)` wrapped in try/except, and build the per-record result list. Return `207 Multi-Status` or `200` with the structured breakdown, depending on API convention.

---

## Security & Auth

### 11. Multi-tenant auth and isolation
**Multi-tenant auth: "Design an auth scheme where different client banks (tenants) hit the same API but must only see their own data. Where do you enforce tenant isolation — dependency, middleware, or DB layer?"**

- Extract tenant identity from the JWT/token (`tenant_id` claim) via a FastAPI dependency, e.g. `get_current_tenant(token: str = Depends(oauth2_scheme))`.
- Enforce isolation at **multiple layers**, defense in depth:
  - **Dependency layer**: reject requests where the path/query `tenant_id` doesn't match the token's tenant.
  - **Service/query layer**: every DB query includes `WHERE tenant_id = :tenant_id` — never trust the client-supplied tenant_id alone.
  - **DB layer** (strongest): use row-level security (Postgres RLS) keyed on a session variable set per-connection from the authenticated tenant, so even a bug in application code can't leak cross-tenant data.
- Avoid relying on isolation enforced only in one layer (e.g., only in the router) — a missed check anywhere becomes a data leak.

---

### 12. OAuth2 refresh tokens without re-validating on every request
**Token refresh under load: "How would you implement OAuth2 with refresh tokens in FastAPI using dependency injection, and how do you avoid re-validating the token on every single request if it's expensive (e.g., calls an external IdP)?"**

- Use short-lived access tokens (JWT, self-contained, signature-verifiable locally — no IdP call needed) and longer-lived refresh tokens.
- Validate the JWT signature and expiry **locally** in a dependency (no network call) for every request — this is fast (just crypto, no I/O).
- Only call the IdP when the access token is expired and the client presents a refresh token to `/token/refresh`.
- Cache IdP public keys (JWKS) locally with periodic refresh, rather than fetching them per request.
- If you need real-time revocation checks, use a fast local cache (Redis) of revoked token IDs rather than hitting the IdP synchronously on every request.

---

### 13. Per-client rate limiting
**Rate limiting a specific client: "One client is hammering your fraud-detection API. How would you implement per-client rate limiting, and where does that logic live in the FastAPI dependency chain?"**


- Implement via a FastAPI dependency that runs early in the chain (before expensive business logic), backed by Redis (`INCR` + `EXPIRE`, or a sliding-window/token-bucket algorithm) keyed by client ID/API key.
- Example: `Depends(rate_limiter)` raises `HTTPException(429)` if the client's request count exceeds their quota in the current window.
- Prefer a library like `slowapi` (FastAPI/Starlette port of Flask-Limiter) or a dedicated API gateway (e.g., Kong, an Nginx layer, or cloud API gateway) for production-grade limiting, since in-process limiting doesn't work correctly across multiple app instances without a shared store like Redis.
- Return standard headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`) so clients can back off correctly.

---

## Resilience & Production Readiness

### 14. Primary fraud model down — fail open, fail closed, or fallback?
**Model failure fallback: "Your primary fraud model service is down. Design the endpoint behavior — do you fail closed, fail open, or fallback to a rules-based model? How does FastAPI's exception handling support this?"**

This is a business/risk decision, not just an engineering one, but the engineering should support all three options:

- **Fail closed** (block/hold the transaction): safest for high-risk transaction types, but hurts customer experience/throughput if the model is flaky.
- **Fail open** (allow the transaction, flag for later review): keeps throughput, but risks letting fraud through — acceptable if you have strong downstream/batch detection.
- **Fallback to a rules-based model**: often the best middle ground — degrade gracefully rather than binary allow/block.

FastAPI implementation: wrap the model call in try/except (or a circuit breaker), catch specific exceptions (timeout, connection error, model-service 5xx), and route to the fallback path. Use custom exception handlers (`@app.exception_handler(ModelUnavailableError)`) to centralize this behavior and keep it consistent across endpoints, and log/alert every time the fallback is triggered so it's visible operationally.

---

### 15. Health check / readiness design for K8s

**Graceful degradation: "Design health check and readiness endpoints for a Kubernetes-deployed FastAPI service that depends on a model artifact store and a feature store. What's the difference between liveness and readiness here?"**

- **Liveness** (`/healthz`): "is the process alive and not deadlocked?" Should be cheap and fast — just confirm the event loop is responsive. If it fails repeatedly, K8s restarts the pod. Don't check downstream dependencies here — a flaky feature store shouldn't cause unnecessary pod restarts.
- **Readiness** (`/readyz`): "can this pod currently serve traffic?" Check that the model artifact is loaded into memory, the feature store connection is healthy, and the DB pool is reachable. If any critical dependency is down, return non-200 so K8s stops routing traffic to this pod (but doesn't kill it) — traffic goes to other healthy pods instead.
- Keep both endpoints excluded from auth middleware and from your regular logging/tracing overhead so they stay lightweight.

---

### 16. Idempotency for a transaction-flagging endpoint

**Idempotency: "A payment/transaction-flagging endpoint might receive duplicate requests due to client retries. How do you design for idempotency in FastAPI (headers, dedup keys, storage)?"**

- Require clients to send an `Idempotency-Key` header (client-generated UUID) with each request.
- On receiving a request, check if that key has been seen before (Redis or DB table: `idempotency_key -> response, status`).
  - If seen and completed: return the **stored response** immediately, don't reprocess.
  - If seen and in-progress: return `409`/`425` or hold, depending on design, to avoid concurrent duplicate processing.
  - If not seen: process normally, then store the key with the response before returning.
- Set a reasonable TTL on stored idempotency keys (e.g., 24h) matching the client's plausible retry window.
- Implement this as a FastAPI dependency/middleware so it's applied consistently across all mutating endpoints, not reimplemented per route.

---

## Testing & Observability

### 17. Testing async endpoints with DB + external API dependencies

**Testing async dependencies: "How would you write tests for an endpoint that depends on an async database session and an external API call, without hitting real services?"**

- Use `pytest-asyncio` with FastAPI's `TestClient` (httpx-based, works with both sync and async apps) or `AsyncClient` for fully async test flows.
- **Override dependencies** using FastAPI's `app.dependency_overrides` dict — swap the real DB session dependency for a test session (e.g., SQLite in-memory or a test Postgres container via `testcontainers`), and swap the external API client for a mock/stub.
- Mock external HTTP calls with `respx` (built for `httpx`) so tests don't hit real services and stay deterministic and fast.
- Use fixtures to reset dependency overrides between tests to avoid test pollution.

---

### 18. Request tracing across microservices

**Request tracing: "You need to trace a request across multiple microservices for debugging a fraud false-positive. How would you propagate a correlation/trace ID through FastAPI middleware and downstream calls?"**

- Generate or extract a **correlation/trace ID** (e.g., from `X-Request-ID` header, or auto-generate if absent) in a FastAPI middleware, and attach it to `request.state.trace_id`.
- Propagate it downstream by adding it to the headers of every outbound `httpx` call.
- Include the trace ID in every log line (structured logging with a logging filter/processor that injects it automatically).
- For proper distributed tracing (not just correlation IDs), integrate OpenTelemetry (`opentelemetry-instrumentation-fastapi`), which auto-instruments FastAPI/Starlette and propagates W3C Trace Context headers automatically across async HTTP calls, exporting spans to Jaeger/Tempo/Datadog etc.

---

### 19. Audit-compliant structured logging without leaking PII or hurting performance

**Structured logging under load: "Design a logging strategy for a FastAPI service where you need to log request/response bodies for audit purposes (compliance) without leaking PII or tanking performance."**

- Log structurally (JSON logs via `structlog` or `python-json-logger`), not free-text strings — makes downstream querying/redaction easier.
- **Redact/mask PII before logging**: implement a redaction layer (regex or field-based) that strips or hashes fields like SSNs, account numbers, full card numbers (keep only last 4) before the log line is emitted — never log raw PII "just in case."
- Log asynchronously (non-blocking log handlers, e.g., writing to a queue that a separate thread/process flushes to disk or a log shipper) so logging I/O doesn't block the event loop under load.
- For compliance audit trails specifically, consider a separate **structured audit log** (distinct from debug/error logs) that captures who/what/when for regulated actions, often written to an append-only/immutable store rather than standard app logs.

---

## Trade-off / Judgment Questions

### 20. Deciding `async def` vs `def`
**Sync vs async decision: "Walk me through how you'd decide whether a given endpoint should be async def or plain def in FastAPI. What's actually happening under the hood with the thread pool if you get it wrong?"**

Decision rule: **if everything inside the function is `await`-able (async DB driver, async HTTP client, no blocking calls), use `async def`. If any part is a blocking/synchronous call (sync DB driver, `requests`, CPU-heavy computation, file I/O without `aiofiles`), use plain `def`.**

Under the hood: Starlette runs `async def` endpoints directly on the single event loop. Plain `def` endpoints are automatically dispatched to a **threadpool** (default size is limited, e.g., 40 threads via `anyio`), so they don't block the event loop but also don't scale infinitely — thousands of concurrent sync requests will exhaust the threadpool and start queuing.

Getting it wrong: declaring `async def` but calling blocking code inside it stalls the *entire* event loop for *every* concurrent request, not just the slow one — this is a classic and serious production bug (looks fine in low-traffic testing, collapses under load).

---

### 21. `BackgroundTasks` vs a full task queue (Celery/RQ)
**Background tasks vs task queue: "When would you use FastAPI's built-in BackgroundTasks versus a full task queue like Celery/RQ? Give a scenario for each."**



**Use `BackgroundTasks`** for lightweight, fire-and-forget work tied to the request lifecycle, running in the same process, where losing the task on a crash/restart is acceptable — e.g., sending a confirmation email, writing an audit log line, invalidating a cache key.

**Use Celery/RQ/arq** when you need:
- Persistence/durability (task survives process restart, backed by a broker like Redis/RabbitMQ),
- Retries with backoff,
- Scheduling (cron-like periodic tasks),
- Scaling workers independently from the API process,
- Visibility into task status/progress (e.g., for the 2-minute report generation scenario in Q4).

Rule of thumb: if the task's failure would be a real business problem (must complete eventually, must be retried, must be tracked), it belongs in a real task queue, not `BackgroundTasks`.

---

### 22. Dependency injection getting too deep (6 layers)
**Dependency injection depth: "Your dependency chain has grown to 6 layers deep (auth → tenant → db session → feature flags → rate limit → business logic). At what point does this become an anti-pattern, and how would you refactor it?"**

Signs it's become an anti-pattern:
- Hard to trace what a given endpoint actually depends on without reading the whole chain.
- Slower request handling if each layer does real work (e.g., separate DB round-trips) rather than being cheap/cached.
- Difficult to test in isolation — mocking 6 layers of dependencies for a unit test is a smell.

Refactor approaches:
- **Consolidate related concerns** into a single composed dependency (e.g., one `AuthContext` dependency that internally resolves auth + tenant + feature flags in one function, rather than 3 separate `Depends()`).
- Use `Depends()` **caching** (FastAPI caches dependency results per-request by default) so shared sub-dependencies aren't recomputed multiple times in the chain.
- Move cross-cutting concerns (rate limiting, logging, request ID) into **middleware** instead of per-route dependencies, since they apply globally rather than being business logic.
- Keep the dependency chain to what's actually needed per route — not every endpoint needs every layer; over-generalizing the chain for "consistency" often causes this bloat.
