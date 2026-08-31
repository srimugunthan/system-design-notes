# FastAPI vs Plain Python: Why It's Worth Using

FastAPI's core value is that it collapses manual boilerplate (routing, validation, serialization, auth, rate limiting, error handling, docs) into type hints and decorators. Below are four side-by-side comparisons covering the most common building blocks of a real API.

---

## 1. Basic REST Endpoint (Create User)

### Without FastAPI (`http.server`)

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import json

class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path != "/users":
            self.send_response(404)
            self.end_headers()
            return

        length = int(self.headers.get("Content-Length", 0))
        raw_body = self.rfile.read(length)

        try:
            data = json.loads(raw_body)
        except json.JSONDecodeError:
            self.send_response(400)
            self.end_headers()
            self.wfile.write(b'{"error": "invalid JSON"}')
            return

        # Manual validation — every field, every type, every rule
        name = data.get("name")
        age = data.get("age")
        email = data.get("email")

        errors = []
        if not isinstance(name, str) or not name:
            errors.append("name must be a non-empty string")
        if not isinstance(age, int) or age < 0:
            errors.append("age must be a non-negative integer")
        if not isinstance(email, str) or "@" not in email:
            errors.append("email must be a valid email")

        if errors:
            self.send_response(422)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            self.wfile.write(json.dumps({"errors": errors}).encode())
            return

        user = {"id": 1, "name": name, "age": age, "email": email}

        self.send_response(201)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(json.dumps(user).encode())

HTTPServer(("localhost", 8000), Handler).serve_forever()
```

### With FastAPI

```python
from fastapi import FastAPI
from pydantic import BaseModel, EmailStr, Field

app = FastAPI()

class UserCreate(BaseModel):
    name: str = Field(min_length=1)
    age: int = Field(ge=0)
    email: EmailStr

class UserOut(UserCreate):
    id: int

@app.post("/users", response_model=UserOut, status_code=201)
async def create_user(user: UserCreate):
    return {"id": 1, **user.model_dump()}
```

### What FastAPI buys you

- **Validation is declarative** — the Pydantic model *is* the validation logic. No `if` chains.
- **Free interactive docs** — `/docs` (Swagger UI) and `/redoc` are generated from type hints.
- **Automatic serialization** — `response_model` controls exactly what shape goes out.
- **Async-native** — `async def` runs on Starlette/ASGI, so I/O-bound work doesn't block the event loop.
- **Editor support** — typed code means autocomplete and type-checkers catch mistakes early.
- **Dependency injection** — `Depends()` centralizes auth, DB sessions, rate limiting, etc.

---

## 2. Authentication (JWT on a Protected Endpoint)

### Without FastAPI

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import json, hmac, hashlib, base64, time

SECRET = b"super-secret-key"

def create_token(user_id):
    header = base64.urlsafe_b64encode(json.dumps({"alg": "HS256"}).encode()).rstrip(b"=")
    payload = base64.urlsafe_b64encode(json.dumps({
        "user_id": user_id, "exp": int(time.time()) + 3600
    }).encode()).rstrip(b"=")
    signing_input = header + b"." + payload
    sig = base64.urlsafe_b64encode(
        hmac.new(SECRET, signing_input, hashlib.sha256).digest()
    ).rstrip(b"=")
    return (signing_input + b"." + sig).decode()

def verify_token(token):
    try:
        header_b64, payload_b64, sig_b64 = token.split(".")
        signing_input = f"{header_b64}.{payload_b64}".encode()
        expected_sig = base64.urlsafe_b64encode(
            hmac.new(SECRET, signing_input, hashlib.sha256).digest()
        ).rstrip(b"=").decode()
        if sig_b64 != expected_sig:
            return None
        payload = json.loads(base64.urlsafe_b64decode(payload_b64 + "=="))
        if payload["exp"] < time.time():
            return None
        return payload
    except Exception:
        return None

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/profile":
            auth_header = self.headers.get("Authorization", "")

            if not auth_header.startswith("Bearer "):
                self.send_response(401)
                self.end_headers()
                self.wfile.write(b'{"error": "missing bearer token"}')
                return

            token = auth_header.removeprefix("Bearer ")
            payload = verify_token(token)

            if payload is None:
                self.send_response(401)
                self.end_headers()
                self.wfile.write(b'{"error": "invalid or expired token"}')
                return

            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            self.wfile.write(json.dumps({"user_id": payload["user_id"]}).encode())

HTTPServer(("localhost", 8000), Handler).serve_forever()
```

Every protected route needs this header-parsing + verification block copy-pasted (or manually called via a helper).

### With FastAPI

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer
import jwt, time

app = FastAPI()
SECRET = "super-secret-key"
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def create_token(user_id: int) -> str:
    return jwt.encode({"user_id": user_id, "exp": time.time() + 3600}, SECRET, algorithm="HS256")

def get_current_user(token: str = Depends(oauth2_scheme)) -> dict:
    try:
        return jwt.decode(token, SECRET, algorithms=["HS256"])
    except jwt.PyJWTError:
        raise HTTPException(status_code=401, detail="invalid or expired token")

@app.get("/profile")
async def read_profile(user: dict = Depends(get_current_user)):
    return {"user_id": user["user_id"]}

@app.post("/token")
async def login(user_id: int):
    return {"access_token": create_token(user_id), "token_type": "bearer"}
```

### What changes

- **`Depends()` replaces manual header parsing** — `get_current_user` is written once and injected into any route needing auth.
- **Auth failures are exceptions, not `if` chains** — `HTTPException` short-circuits with the right status code and body.
- **`OAuth2PasswordBearer` documents itself** — `/docs` shows lock icons and an "Authorize" button.
- **Composable security** — stack dependencies (`get_current_user` → `get_current_active_user` → `require_admin`) instead of nested conditionals.
- **Same pattern extends** to API keys, OAuth2 flows, or scopes via `APIKeyHeader`, `OAuth2PasswordRequestForm`, `Security()`.

---

## 3. Rate Limiting

### Without FastAPI

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import json, time
from collections import defaultdict, deque

request_log = defaultdict(deque)
LIMIT = 5
WINDOW = 60

def is_rate_limited(client_id):
    now = time.time()
    log = request_log[client_id]

    while log and log[0] < now - WINDOW:
        log.popleft()

    if len(log) >= LIMIT:
        return True

    log.append(now)
    return False

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/data":
            client_id = self.client_address[0]  # crude — IP as identity

            if is_rate_limited(client_id):
                self.send_response(429)
                self.send_header("Content-Type", "application/json")
                self.send_header("Retry-After", str(WINDOW))
                self.end_headers()
                self.wfile.write(b'{"error": "rate limit exceeded"}')
                return

            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            self.wfile.write(json.dumps({"data": "here you go"}).encode())

HTTPServer(("localhost", 8000), Handler).serve_forever()
```

Problems: no thread safety (race conditions on the deque), single-process only, and every route needs the check pasted in manually.

### With FastAPI (using `slowapi`)

```python
from fastapi import FastAPI, Request
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/data")
@limiter.limit("5/minute")
async def get_data(request: Request):
    return {"data": "here you go"}

@app.get("/expensive-inference")
@limiter.limit("2/minute")   # different limit, same pattern
async def run_inference(request: Request):
    return {"result": "..."}
```

### What changes

- **Declarative limits** — `"5/minute"` replaces the deque and manual eviction loop.
- **Per-route granularity is trivial** — each endpoint gets its own `@limiter.limit(...)`.
- **429s and headers handled for you** — correct status, `Retry-After`, and JSON body come from `_rate_limit_exceeded_handler`.
- **Pluggable backend** — swap in-memory for Redis (`storage_uri="redis://..."`) with one line — critical once you run multiple worker processes/pods.
- **Composable with auth** — `key_func` can switch from IP-based to identity-based (`request.state.user_id`) for per-user limits.

---

## 4. Error Handling

### Without FastAPI

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import json, traceback

class InsufficientFundsError(Exception):
    def __init__(self, balance, requested):
        self.balance = balance
        self.requested = requested

class UserNotFoundError(Exception):
    def __init__(self, user_id):
        self.user_id = user_id

def get_user(user_id):
    if user_id != 1:
        raise UserNotFoundError(user_id)
    return {"id": 1, "balance": 100}

def withdraw(user, amount):
    if amount > user["balance"]:
        raise InsufficientFundsError(user["balance"], amount)
    return {"new_balance": user["balance"] - amount}

class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path == "/withdraw":
            length = int(self.headers.get("Content-Length", 0))
            data = json.loads(self.rfile.read(length))

            try:
                user = get_user(data["user_id"])
                result = withdraw(user, data["amount"])

                self.send_response(200)
                self.send_header("Content-Type", "application/json")
                self.end_headers()
                self.wfile.write(json.dumps(result).encode())

            except UserNotFoundError as e:
                self.send_response(404)
                self.send_header("Content-Type", "application/json")
                self.end_headers()
                self.wfile.write(json.dumps({
                    "error": "user_not_found", "user_id": e.user_id
                }).encode())

            except InsufficientFundsError as e:
                self.send_response(422)
                self.send_header("Content-Type", "application/json")
                self.end_headers()
                self.wfile.write(json.dumps({
                    "error": "insufficient_funds",
                    "balance": e.balance, "requested": e.requested
                }).encode())

            except KeyError as e:
                self.send_response(400)
                self.end_headers()
                self.wfile.write(json.dumps({"error": f"missing field {e}"}).encode())

            except Exception:
                traceback.print_exc()
                self.send_response(500)
                self.end_headers()
                self.wfile.write(json.dumps({"error": "internal server error"}).encode())

HTTPServer(("localhost", 8000), Handler).serve_forever()
```

Every route re-implements the same try/except ladder. Miss one exception type and it either 500s ungracefully or leaks a raw traceback.

### With FastAPI

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI()

class InsufficientFundsError(Exception):
    def __init__(self, balance: float, requested: float):
        self.balance = balance
        self.requested = requested

class UserNotFoundError(Exception):
    def __init__(self, user_id: int):
        self.user_id = user_id

# Registered ONCE, applies to every route automatically
@app.exception_handler(UserNotFoundError)
async def user_not_found_handler(request: Request, exc: UserNotFoundError):
    return JSONResponse(
        status_code=404,
        content={"error": "user_not_found", "user_id": exc.user_id},
    )

@app.exception_handler(InsufficientFundsError)
async def insufficient_funds_handler(request: Request, exc: InsufficientFundsError):
    return JSONResponse(
        status_code=422,
        content={"error": "insufficient_funds", "balance": exc.balance, "requested": exc.requested},
    )

class WithdrawRequest(BaseModel):
    user_id: int
    amount: float

def get_user(user_id: int):
    if user_id != 1:
        raise UserNotFoundError(user_id)
    return {"id": 1, "balance": 100}

@app.post("/withdraw")
async def withdraw(req: WithdrawRequest):
    user = get_user(req.user_id)
    if req.amount > user["balance"]:
        raise InsufficientFundsError(user["balance"], req.amount)
    return {"new_balance": user["balance"] - req.amount}
```

No `try/except` in the route at all — it just raises the domain exception and walks away.

### What changes

- **Handlers are registered globally, not per-route** — `@app.exception_handler(...)` applies everywhere automatically.
- **Invalid fields are caught before your code runs** — Pydantic validation replaces manual `KeyError`/type checks.
- **500s are safe by default** — uncaught exceptions never leak a traceback to the client.
- **Business logic and error formatting are separated** — routes stay readable as pure logic.
- **Validation errors already have a built-in handler** — FastAPI auto-returns a structured 422 for Pydantic failures.

---

## Summary

| Concern | Plain Python | FastAPI |
|---|---|---|
| Validation | Manual `if`/`isinstance` chains | Pydantic models, declarative |
| Docs | None, maintained separately | Auto-generated `/docs`, `/redoc` |
| Auth | Copy-pasted verify blocks per route | `Depends()` dependency chain |
| Rate limiting | Manual deque + eviction logic, no shared state across processes | `@limiter.limit(...)`, pluggable Redis backend |
| Error handling | Per-route try/except ladders | Global `@app.exception_handler(...)` |
| Async | Manual (or absent) | Native `async def` on ASGI |
| Type safety | None | Full editor/type-checker support |

**Relevance to your work:** for services like AuditAgent, Shield-Fin, or a fraud-scoring endpoint, the payoff compounds — centralized auth dependencies, Redis-backed rate limits across replicas, and named exception handlers all reduce the amount of repeated defensive code as the number of endpoints and failure modes grows.
