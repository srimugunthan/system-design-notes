# Designing a FastAPI ML Prediction Service: Auth, Logging, and Error Handling

Design for a REST API that authenticates requests via API key, logs every request with latency, and returns structured error responses — walked through layer by layer with a block diagram, followed by the implementation for each piece.

## Block diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                              Client                                  │
│                    (sends X-API-Key header + payload)                │
└───────────────────────────────┬───────────────────────────────────-─┘
                                 │ HTTP request
                                 ▼
┌───────────────────────────────────────────────────────────────────-─┐
│                         Uvicorn / ASGI server                        │
└───────────────────────────────┬────────────────────────────────────-┘
                                 ▼
┌───────────────────────────────────────────────────────────────────-─┐
│  MIDDLEWARE LAYER (runs for every request, in order)                 │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 1. RequestLoggingMiddleware                                   │   │
│  │    - start_time = now()                                       │   │
│  │    - call_next(request)                                       │   │
│  │    - latency = now() - start_time                             │   │
│  │    - log: method, path, status_code, latency, request_id      │   │
│  └─────────────────────────────────────────────────────────────┘    │
└───────────────────────────────┬────────────────────────────────────-┘
                                 ▼
┌───────────────────────────────────────────────────────────────────-─┐
│  ROUTING LAYER (APIRouter per resource)                              │
│  ┌───────────────────┐  ┌───────────────────┐  ┌──────────────────┐ │
│  │  /health (router)  │  │ /v1/predict        │  │ /v1/models        │
│  │  no auth dep       │  │ (router)            │  │ (router)          │
│  └───────────────────┘  │ Depends(verify_key) │  │ Depends(verify_key)│
│                          └─────────┬───────────┘  └─────────┬────────┘│
└────────────────────────────────────┼──────────────────────-┼─────────┘
                                      ▼                       ▼
┌───────────────────────────────────────────────────────────────────-─┐
│  DEPENDENCY LAYER                                                    │
│  ┌─────────────────────────┐   ┌───────────────────────────────┐    │
│  │ verify_api_key()          │   │ get_model()                    │  │
│  │ - reads X-API-Key header  │   │ - loads/returns cached          │ │
│  │ - checks against store    │   │   model instance                │ │
│  │ - raises 401 if invalid   │   └───────────────────────────────┘  │
│  └─────────────────────────┘                                        │
└───────────────────────────────┬─────────────────────────────────────┘
                                 ▼
┌───────────────────────────────────────────────────────────────────-─┐
│  BUSINESS LOGIC (route handler)                                      │
│  - validate payload (Pydantic model)                                 │
│  - run model.predict(payload)                                        │
│  - may raise domain exceptions:                                      │
│      ModelUnavailableError, InvalidInputError, PredictionTimeoutError│
└───────────────────────────────┬────────────────────────────────────-┘
                                 ▼
┌───────────────────────────────────────────────────────────────────-─┐
│  EXCEPTION HANDLING LAYER (registered globally on `app`)             │
│  ┌───────────────────────┐ ┌───────────────────────┐ ┌─────────────┐│
│  │ RequestValidationError │ │ Domain exceptions      │ │ Exception    │
│  │  → 422 structured body │ │  → mapped status codes │ │  → 500 safe  │
│  └───────────────────────┘ └───────────────────────┘ └─────────────┘│
│  All produce the SAME response shape: {error, detail, request_id}    │
└───────────────────────────────┬────────────────────────────────────-┘
                                 ▼
┌───────────────────────────────────────────────────────────────────-─┐
│                   Structured JSON response to client                │
└───────────────────────────────────────────────────────────────────-─┘
```

The key design principle: **each concern lives in exactly one layer**, and layers only talk to the next one down. Routes never parse headers manually, never catch generic exceptions, never write logs directly — they raise and return, and the layers around them handle the rest.

## 1. Project structure

```
app/
├── main.py              # app instance, middleware, exception handlers, router includes
├── config.py             # API key store, settings
├── dependencies.py       # verify_api_key, get_model
├── schemas.py             # Pydantic request/response models
├── exceptions.py           # domain exception classes
├── logging_middleware.py    # latency + request logging
└── routers/
    ├── health.py
    └── predict.py
```

## 2. Structured error contract (defined first — everything else maps into this)

```python
# schemas.py
from pydantic import BaseModel

class ErrorResponse(BaseModel):
    error: str          # machine-readable code, e.g. "invalid_api_key"
    detail: str          # human-readable message
    request_id: str       # ties the error back to a log line
```

Every failure path — auth, validation, business logic, unexpected crash — returns this same shape. That consistency is what makes the API predictable for clients.

## 3. Domain exceptions

```python
# exceptions.py
class ModelUnavailableError(Exception):
    def __init__(self, model_name: str):
        self.model_name = model_name

class InvalidInputError(Exception):
    def __init__(self, reason: str):
        self.reason = reason

class PredictionTimeoutError(Exception):
    pass
```

These carry only the *data* needed to build a response — no HTTP knowledge. That separation means the prediction logic stays testable without importing FastAPI at all.

## 4. API key authentication (dependency)

```python
# dependencies.py
from fastapi import Header, HTTPException
from app.config import VALID_API_KEYS

async def verify_api_key(x_api_key: str = Header(...)) -> str:
    if x_api_key not in VALID_API_KEYS:
        raise HTTPException(status_code=401, detail="invalid or missing API key")
    return x_api_key
```

Attached once per router (not per route) so every prediction endpoint is covered automatically, with zero repetition:

```python
# routers/predict.py
from fastapi import APIRouter, Depends
from app.dependencies import verify_api_key

router = APIRouter(
    prefix="/v1",
    tags=["predict"],
    dependencies=[Depends(verify_api_key)],
)
```

## 5. Request logging with latency (middleware)

Middleware is the right layer here — not a dependency — because it needs to wrap the *entire* request/response cycle, including time spent in routing and exception handling, and it should apply uniformly without being declared per-route.

```python
# logging_middleware.py
import time, uuid, logging
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

logger = logging.getLogger("api.requests")

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = str(uuid.uuid4())
        request.state.request_id = request_id  # available to handlers/exception handlers

        start = time.perf_counter()
        response = await call_next(request)
        latency_ms = (time.perf_counter() - start) * 1000

        logger.info(
            "request_id=%s method=%s path=%s status=%d latency_ms=%.2f",
            request_id, request.method, request.url.path,
            response.status_code, latency_ms,
        )
        response.headers["X-Request-ID"] = request_id
        return response
```

Storing `request_id` on `request.state` is what lets the exception handlers below include it in the error body — so a client-reported error and a server log line can be correlated directly.

## 6. Global exception handlers

```python
# main.py (excerpt)
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from app.exceptions import ModelUnavailableError, InvalidInputError, PredictionTimeoutError

def _error(request: Request, code: str, detail: str):
    return {
        "error": code,
        "detail": detail,
        "request_id": getattr(request.state, "request_id", "unknown"),
    }

async def validation_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(status_code=422, content=_error(request, "validation_error", str(exc.errors())))

async def model_unavailable_handler(request: Request, exc: ModelUnavailableError):
    return JSONResponse(status_code=503, content=_error(request, "model_unavailable", f"model '{exc.model_name}' is not loaded"))

async def invalid_input_handler(request: Request, exc: InvalidInputError):
    return JSONResponse(status_code=400, content=_error(request, "invalid_input", exc.reason))

async def timeout_handler(request: Request, exc: PredictionTimeoutError):
    return JSONResponse(status_code=504, content=_error(request, "prediction_timeout", "model did not respond in time"))

async def unhandled_handler(request: Request, exc: Exception):
    # log full traceback server-side; never leak it to the client
    logger.exception("unhandled error request_id=%s", getattr(request.state, "request_id", "unknown"))
    return JSONResponse(status_code=500, content=_error(request, "internal_error", "an unexpected error occurred"))
```

## 7. Wiring it all together

```python
# main.py
from fastapi import FastAPI
from app.logging_middleware import RequestLoggingMiddleware
from app.routers import health, predict
from app.exceptions import ModelUnavailableError, InvalidInputError, PredictionTimeoutError
from fastapi.exceptions import RequestValidationError

app = FastAPI(title="ML Prediction Service")

# Middleware — order matters; this wraps everything below it
app.add_middleware(RequestLoggingMiddleware)

# Exception handlers — registered globally, apply to every router
app.add_exception_handler(RequestValidationError, validation_handler)
app.add_exception_handler(ModelUnavailableError, model_unavailable_handler)
app.add_exception_handler(InvalidInputError, invalid_input_handler)
app.add_exception_handler(PredictionTimeoutError, timeout_handler)
app.add_exception_handler(Exception, unhandled_handler)

# Routers — auth dependency lives inside predict.router, not repeated here
app.include_router(health.router)
app.include_router(predict.router)
```

## 8. The route handler itself — stays thin

```python
# routers/predict.py (continued)
from fastapi import Depends
from app.schemas import PredictRequest, PredictResponse
from app.dependencies import get_model
from app.exceptions import InvalidInputError

@router.post("/predict", response_model=PredictResponse)
async def predict(payload: PredictRequest, model=Depends(get_model)):
    if payload.features is None or len(payload.features) == 0:
        raise InvalidInputError("features array cannot be empty")

    prediction = await model.predict_async(payload.features)
    return PredictResponse(prediction=prediction, model_version=model.version)
```

Notice what's *not* here: no try/except, no manual header parsing, no logging call, no status-code decisions. The handler only expresses domain logic — everything else is handled by the layers around it.

## Why this shape holds up in production

- **Auth is enforced structurally** — a new prediction route added under `predict.router` gets the API key check automatically; it's not possible to forget it on one endpoint.
- **Latency logging is uniform** — because it's middleware wrapping `call_next`, it captures total time including exception handling, not just the handler body.
- **Errors are consistent** — a client can always parse `{error, detail, request_id}` regardless of whether it was an auth failure, a bad payload, or a model crash — no branch logic needed on the client side.
- **`request_id` ties logs to responses** — when a client reports "I got a 503," you grep the log for that ID and see exactly what happened, with the latency at the time.
- **Testable in isolation** — `get_model` and `verify_api_key` can both be swapped via `app.dependency_overrides` in tests, so you can test the routing/error-handling contract without a real model loaded.
