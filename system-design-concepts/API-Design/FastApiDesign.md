# API Design with FastAPI

> A practical guide to routing, middleware, authentication, and error handling in a cohesive ML prediction service.

---

## Part 1 — Building the Service

### Architecture Overview

The service is organized into three layers that work together:

- **Middleware** — intercepts every request before and after routing
- **Router** — handles endpoint logic with dependency injection
- **Exception Handlers** — catch and format all errors uniformly

---

### 1. Project Structure

```
ml_service/
├── main.py          # App factory, middleware, exception handlers
├── router.py        # Prediction endpoints
├── dependencies.py  # API key auth dependency
├── schemas.py       # Request/response models
└── model.py         # ML model wrapper
```

---

### 2. Schemas — Define Your Contracts First

Separating schemas from routing keeps validation logic centralized and reusable across endpoints.

```python
# schemas.py
from pydantic import BaseModel, Field
from typing import Any
import time

class PredictionRequest(BaseModel):
    features: list[float] = Field(..., min_length=1)
    model_version: str = Field(default="v1")

class PredictionResponse(BaseModel):
    prediction: Any
    model_version: str
    latency_ms: float
    request_id: str

class ErrorResponse(BaseModel):
    error_code: str
    message: str
    request_id: str
    timestamp: float = Field(default_factory=time.time)
```

---

### 3. Authentication Dependency

`secrets.compare_digest` is critical here — a naive `==` comparison leaks information about key validity through timing differences.

```python
# dependencies.py
from fastapi import Security, HTTPException, status
from fastapi.security import APIKeyHeader
import secrets

API_KEY_HEADER = APIKeyHeader(name="X-API-Key", auto_error=False)
VALID_API_KEYS = {"sk-prod-abc123", "sk-dev-xyz789"}

async def verify_api_key(api_key: str = Security(API_KEY_HEADER)) -> str:
    if not api_key:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail={"error_code": "MISSING_API_KEY", "message": "X-API-Key header is required"}
        )
    valid = any(secrets.compare_digest(api_key, k) for k in VALID_API_KEYS)
    if not valid:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail={"error_code": "INVALID_API_KEY", "message": "Provided API key is not authorized"}
        )
    return api_key
```

---

### 4. Router — Prediction Endpoints

The `request_id` is read from `request.state`, which the middleware populates — keeping concerns cleanly separated.

```python
# router.py
from fastapi import APIRouter, Depends, Request
from schemas import PredictionRequest, PredictionResponse
from dependencies import verify_api_key
from model import run_inference
import time, uuid

router = APIRouter(prefix="/api/v1", tags=["predictions"])

@router.post(
    "/predict",
    response_model=PredictionResponse,
    dependencies=[Depends(verify_api_key)],
)
async def predict(request: Request, payload: PredictionRequest):
    start = time.perf_counter()
    request_id = request.state.request_id

    result = await run_inference(payload.features, payload.model_version)

    latency_ms = (time.perf_counter() - start) * 1000
    return PredictionResponse(
        prediction=result,
        model_version=payload.model_version,
        latency_ms=latency_ms,
        request_id=request_id,
    )

@router.get("/health")  # No auth — public health check
async def health():
    return {"status": "ok"}
```

---

### 5. Middleware — Logging + Request ID Injection

The middleware wraps `call_next` in a try/except so latency is always logged — even for failed requests.

```python
# main.py (middleware section)
import logging, time, uuid
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware

logger = logging.getLogger("ml_service")

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = str(uuid.uuid4())
        request.state.request_id = request_id

        start = time.perf_counter()
        try:
            response: Response = await call_next(request)
            latency_ms = (time.perf_counter() - start) * 1000
            logger.info(
                f"request_id={request_id} method={request.method} "
                f"path={request.url.path} status={response.status_code} "
                f"latency_ms={latency_ms:.2f}"
            )
            response.headers["X-Request-ID"] = request_id
            return response
        except Exception as exc:
            latency_ms = (time.perf_counter() - start) * 1000
            logger.error(f"request_id={request_id} path={request.url.path} error={exc} latency_ms={latency_ms:.2f}")
            raise
```

---

### 6. Exception Handlers — Unified Error Shape

All three handlers share the same response shape — clients can always rely on `error_code`, `message`, and `request_id` being present.

```python
# main.py (exception handlers section)
from fastapi import FastAPI, HTTPException
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

def add_exception_handlers(app: FastAPI):

    @app.exception_handler(RequestValidationError)
    async def validation_error_handler(request, exc):
        return JSONResponse(status_code=422, content={
            "error_code": "VALIDATION_ERROR",
            "message": str(exc.errors()),
            "request_id": getattr(request.state, "request_id", "unknown"),
            "timestamp": time.time(),
        })

    @app.exception_handler(HTTPException)
    async def http_error_handler(request, exc):
        detail = exc.detail if isinstance(exc.detail, dict) else {"message": exc.detail}
        return JSONResponse(status_code=exc.status_code, content={
            **detail,
            "request_id": getattr(request.state, "request_id", "unknown"),
            "timestamp": time.time(),
        })

    @app.exception_handler(Exception)
    async def generic_error_handler(request, exc):
        logger.exception(f"Unhandled error for request_id={getattr(request.state, 'request_id', '?')}")
        return JSONResponse(status_code=500, content={
            "error_code": "INTERNAL_ERROR",
            "message": "An unexpected error occurred. Please try again.",
            "request_id": getattr(request.state, "request_id", "unknown"),
            "timestamp": time.time(),
        })
```

---

### 7. App Factory — Wire Everything Together

```python
# main.py
from fastapi import FastAPI
from router import router

def create_app() -> FastAPI:
    app = FastAPI(title="ML Prediction Service", version="1.0.0")
    app.add_middleware(RequestLoggingMiddleware)
    app.include_router(router)
    add_exception_handlers(app)
    return app

app = create_app()
```

---

### Request Lifecycle

```
Incoming Request
      │
      ▼
RequestLoggingMiddleware          ← assigns request_id, starts timer
      │
      ▼
Auth Dependency (verify_api_key)  ← 401/403 if invalid
      │
      ▼
Route Handler (predict)           ← business logic + inference
      │
      ▼
RequestLoggingMiddleware (resume) ← logs latency + status code
      │
      ▼
Response (with X-Request-ID header)
```

If anything throws, the exception handlers intercept before the middleware finishes logging, ensuring every failure is both logged and formatted consistently.

---

## Part 2 — Advantages of This Design

### Separation of Concerns is Enforced Structurally

Auth lives in `dependencies.py`, logging in middleware, error formatting in exception handlers, and business logic in `router.py`. Each layer can be tested, replaced, or scaled independently. When your model serving logic changes, you don't touch the auth code.

### Request Tracing is First-Class

The `request_id` flows through every layer — middleware, route handlers, error responses, and logs. In production with distributed systems, this is what lets you grep a single ID across log aggregators like Datadog or ELK and reconstruct an entire request's journey.

### Errors are Client-Friendly by Design

Every failure — validation, auth, unhandled exception — returns the same shape: `error_code`, `message`, `request_id`, `timestamp`. Clients write one error-handling path. This is particularly valuable for ML services where downstream consumers (pipelines, dashboards, other services) need predictable failure contracts.

### Latency is Always Captured

Because the timer lives in middleware rather than the route handler, you measure the full roundtrip — including auth overhead and serialization. A timer inside the route handler would silently miss latency from failed auth or Pydantic validation.

### Dependency Injection Makes Testing Clean

You can override `verify_api_key` in tests with a no-op, swap the model with a mock, and test routing logic in complete isolation — without spinning up a real server or touching environment variables.

---

## Part 3 — What FastAPI Specifically Brings

### Pydantic Integration is Automatic

Request bodies are validated against your schema before your handler is ever called. Invalid input never reaches your ML model. Without FastAPI, you'd write this validation manually for every endpoint.

### Dependency Injection is a First-Class Primitive

`Depends()` lets you compose auth, database connections, and feature flags declaratively at the route level. In Flask or raw WSGI, you'd use decorators or global middleware, which are harder to scope per-route and harder to override in tests.

### `request.state` for Cross-Layer Data Sharing

The middleware injects `request_id` there, and the route handler reads it without any coupling between the two. Flask has `g` which is similar but request-scoped only within the same thread — it gets awkward with async.

### Async is Native

`async def` handlers and middleware work out of the box. For ML inference that calls external model servers (TorchServe, Triton, SageMaker endpoints), true async matters — you can serve other requests while awaiting inference instead of blocking a thread.

### Exception Handlers are Registered Per Type

FastAPI lets you attach different handlers to `RequestValidationError`, `HTTPException`, and `Exception` separately. This is cleaner than a single catch-all middleware that branches on `isinstance`.

### OpenAPI Docs are Generated Automatically

Your `PredictionRequest` and `PredictionResponse` schemas automatically appear in `/docs`. For an ML service, this is non-trivial — stakeholders and consumers can see exactly what the API expects without reading code.

---

## Part 4 — Can You Do This Without FastAPI?

Yes — entirely possible. Here's how each piece maps to alternatives.

### Flask + Extensions

The most common alternative. You'd use `flask-restx` or `marshmallow` for schema validation, `before_request` / `after_request` hooks for middleware-like behavior, and `@app.errorhandler` for exception handling. The main cost is that none of it is integrated — you wire the pieces together manually, and async support is bolted on rather than native.

### Raw Starlette

What FastAPI is built on. Gives you everything except Pydantic integration and dependency injection. Routing, middleware, and exception handlers all work the same way. If you find FastAPI too opinionated, Starlette is the natural step down.

### Django REST Framework

Viable for larger teams that want batteries-included — authentication backends, throttling, serializers. It's heavier and more opinionated, and async support is still maturing. It makes more sense for CRUD-heavy services than ML prediction endpoints.

### Pure ASGI

You could hand-write everything: parse JSON with `json.loads`, validate manually, write a middleware class that wraps the ASGI callable. This is what frameworks do under the hood. Instructive to do once, but not practical at scale.

---

### Framework Comparison

| Feature               | FastAPI              | Flask                  | DRF                |
|-----------------------|----------------------|------------------------|--------------------|
| Async Native          | ✓                    | Partial                | Partial            |
| Schema Validation     | Pydantic (auto)      | Manual / marshmallow   | Serializers        |
| Dependency Injection  | Built-in             | Manual decorators      | Limited            |
| OpenAPI Docs          | Auto-generated       | Plugin needed          | Plugin needed      |
| Exception Handlers    | Per-type             | Per-code               | Per-code           |
| Best For              | ML / async APIs      | Simple services        | CRUD apps          |

---

> **Key Insight:** The design pattern — middleware for cross-cutting concerns, dependency injection for auth, typed schemas for validation, uniform error shapes — is framework-agnostic. FastAPI just makes each of those patterns the path of least resistance rather than something you construct yourself.
