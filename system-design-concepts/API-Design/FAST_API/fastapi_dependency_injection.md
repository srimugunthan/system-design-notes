# Dependency Injection in FastAPI

Dependency injection (DI) is one of FastAPI's more distinctive features — it's how you share logic (auth, DB connections, config, common query params) across routes without repeating yourself or manually wiring things together.

## The core idea

A "dependency" is just a callable (function or class) that FastAPI calls *for you* before running your route, and whose return value gets injected as a parameter. You declare it with `Depends()`.

```python
from fastapi import FastAPI, Depends

app = FastAPI()

def get_query_params(q: str = "", limit: int = 10):
    return {"q": q, "limit": limit}

@app.get("/items")
async def read_items(params: dict = Depends(get_query_params)):
    return params
```

Here, FastAPI sees `Depends(get_query_params)`, calls `get_query_params()` itself (extracting `q` and `limit` from the query string), and hands you the result. You never call it directly.

## Why this matters: reuse across routes

The real value shows up once multiple routes need the same setup — a DB session is the classic example.

```python
from fastapi import FastAPI, Depends
from sqlalchemy.orm import Session

def get_db():
    db = SessionLocal()
    try:
        yield db          # this is what gets injected
    finally:
        db.close()         # runs after the request finishes, even on error

@app.get("/users/{user_id}")
async def read_user(user_id: int, db: Session = Depends(get_db)):
    return db.query(User).filter(User.id == user_id).first()

@app.post("/users")
async def create_user(user: UserCreate, db: Session = Depends(get_db)):
    db_user = User(**user.model_dump())
    db.add(db_user)
    db.commit()
    return db_user
```

Every route that needs a DB session just adds `db: Session = Depends(get_db)` — the connect/close lifecycle is written once. Note the `yield` — FastAPI treats this like a context manager: code before `yield` runs before the route, code after runs afterward (even if the route raises an exception), which is how cleanup gets guaranteed.

## Dependencies can depend on other dependencies

This is where it becomes genuinely powerful — you can chain them, like the auth layering from earlier:

```python
def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    user = decode_token(token)
    if not user:
        raise HTTPException(401, "invalid token")
    return user

def get_current_active_user(user: User = Depends(get_current_user)) -> User:
    if not user.is_active:
        raise HTTPException(400, "inactive user")
    return user

def require_admin(user: User = Depends(get_current_active_user)) -> User:
    if not user.is_admin:
        raise HTTPException(403, "admin only")
    return user

@app.delete("/users/{user_id}")
async def delete_user(user_id: int, admin: User = Depends(require_admin)):
    ...
```

`require_admin` builds on `get_current_active_user`, which builds on `get_current_user`. FastAPI resolves the whole chain automatically — and if the same dependency is needed by two parameters in one request, it's only *called once per request* (cached, not re-run) unless you explicitly disable that.

## Dependencies that apply to a whole router, not just one route

You don't have to attach `Depends()` per-parameter — you can apply it to an entire route or router when you don't need its return value, just its side effect (e.g., an auth check):

```python
from fastapi import APIRouter, Depends

router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(require_admin)]  # applies to every route below
)

@router.get("/stats")
async def stats():
    return {"users": 1000}

@router.delete("/purge")
async def purge():
    ...
```

Both `/admin/stats` and `/admin/purge` now require admin auth, with zero repetition.

## Class-based dependencies

For dependencies that need configuration, a class with `__call__` works well:

```python
class RateLimiter:
    def __init__(self, max_calls: int, window: int):
        self.max_calls = max_calls
        self.window = window

    def __call__(self, request: Request):
        # check/update counter using self.max_calls, self.window
        ...

check_strict = RateLimiter(max_calls=5, window=60)
check_relaxed = RateLimiter(max_calls=100, window=60)

@app.get("/expensive")
async def expensive(limit: None = Depends(check_strict)):
    ...

@app.get("/cheap")
async def cheap(limit: None = Depends(check_relaxed)):
    ...
```

Same dependency function, different instances with different config — not possible with a plain function unless you use closures or partials.

## Why this is genuinely different from Flask-style decorators

In Flask, cross-cutting concerns are usually handled with decorators (`@login_required`) or middleware — both are somewhat opaque (you can't easily tell *what* they inject or depend on without reading their internals). FastAPI's DI is:

- **Declared in the function signature**, so it's visible and type-checked at the call site.
- **Composable** — dependencies chain into other dependencies naturally.
- **Documented automatically** — if a dependency declares query params, headers, or security schemes, they show up correctly in `/docs`.
- **Testable in isolation** — you can override any dependency in tests via `app.dependency_overrides`, without touching route code:

```python
def fake_get_db():
    yield TestSessionLocal()

app.dependency_overrides[get_db] = fake_get_db  # swap real DB for test DB
```

For work like AuditAgent, Shield-Fin, or fraud scoring, this last point matters a lot in practice: DI lets you swap real auth/DB/LLM-client dependencies for mocks in tests without touching a single route function.
