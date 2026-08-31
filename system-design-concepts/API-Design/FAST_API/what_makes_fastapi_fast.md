# What Makes FastAPI "Fast"?

"Fast" in FastAPI actually refers to two different things — runtime performance *and* development speed. Here's what drives each.

## 1. Runtime speed: it's built on ASGI, not WSGI

Traditional frameworks (Flask, Django by default) use WSGI — one thread/process blocks per request until it finishes. FastAPI sits on **Starlette**, which runs on **ASGI**, served by **Uvicorn** (using `uvloop` and `httptools` under the hood — both written in C/Cython). This means a single worker can juggle many concurrent requests without blocking, as long as your I/O is `async`.

This is the part that actually matters for throughput:

```python
import httpx
from fastapi import FastAPI

app = FastAPI()

# BLOCKING — ties up the worker for the full duration of the call
@app.get("/sync-call")
def sync_call():
    import requests
    r = requests.get("https://api.example.com/slow-endpoint")  # blocks event loop
    return r.json()

# NON-BLOCKING — worker is free to handle other requests while waiting
@app.get("/async-call")
async def async_call():
    async with httpx.AsyncClient() as client:
        r = await client.get("https://api.example.com/slow-endpoint")
    return r.json()
```

If that endpoint takes 500ms and you get 50 concurrent requests:

- **`sync_call`** on a single worker processes them essentially one at a time (~25 seconds total), because the `requests` call blocks the whole event loop.
- **`async_call`** lets Uvicorn interleave all 50 while each is waiting on the network — they mostly overlap, so total time is close to ~500ms–1s, not 25s.

This is why FastAPI benchmarks close to Node.js/Go frameworks on I/O-bound workloads (TechEmpower benchmarks are the usual reference) — the speed comes from Starlette/Uvicorn, not from FastAPI itself doing anything special.

## 2. Validation speed: Pydantic v2's Rust core

Pydantic v2 (which FastAPI uses for request/response models) rewrote its validation core in Rust (`pydantic-core`), replacing pure-Python validation from v1. Parsing and validating a JSON body into a typed model is now compiled code, not interpreted Python loops:

```python
from pydantic import BaseModel

class Transaction(BaseModel):
    amount: float
    currency: str
    account_id: int

# This validation call runs through pydantic-core (Rust), not Python
txn = Transaction.model_validate({"amount": 500.0, "currency": "USD", "account_id": 42})
```

For something like fraud scoring where you're validating high volumes of transaction payloads, this is a real, measurable difference over v1 or hand-written validation.

## 3. Development speed (the other "fast")

This is the boilerplate reduction covered in earlier comparisons — type hints replace manual validation, routing, and doc-writing. It's "fast" as in *time-to-ship*, not requests/second, but it's the meaning most people actually associate with the name.

## The caveat

Async only helps if your I/O is actually async. If you write `async def` but call a blocking library inside it (like `requests` instead of `httpx`, or a sync DB driver), you block the event loop anyway and lose the benefit — worse, you block it for *every* concurrent request on that worker, not just your own. For CPU-bound work (e.g. running a local model inference, not calling an external API), async doesn't help either — that needs a thread/process pool (`run_in_threadpool`, or a separate worker) regardless of framework.
