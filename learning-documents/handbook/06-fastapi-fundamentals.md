# 06 — FastAPI: How It Actually Works ⭐

[← 05 RAG](05-rag-and-retrieval.md) · [Index](00-INDEX.md) · Next: [07 Auth →](07-fastapi-auth.md)

FastAPI is the default Python framework for AI backends. It's still pre-1.0 (0.13x+ in 2026, verify), so pin versions. Minor releases can contain breaking changes.

---

## 1. The stack under FastAPI

```
HTTP client
   │
Uvicorn / Hypercorn / Granian   ← ASGI server: sockets, HTTP parsing, event loop
   │  (ASGI interface: async def app(scope, receive, send))
Starlette                       ← routing, middleware, requests/responses, WebSockets, background tasks
   │
FastAPI                         ← type hints → validation, dependency injection, OpenAPI docs
   │
Pydantic v2 (Rust core)         ← parsing/validation/serialization
```

- **WSGI** (Flask, classic Django) = one request per thread, synchronous.
- **ASGI** = async. One process handles thousands of concurrent connections on an event loop. That's important for LLM apps, which spend most of their time *waiting* on model APIs and streaming tokens.

---

## 2. Minimal app, explained

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, Field

app = FastAPI(title="Orders API")

class OrderIn(BaseModel):
    item: str = Field(min_length=1, max_length=50)
    qty: int = Field(gt=0, le=100)

class OrderOut(BaseModel):
    id: int
    item: str
    qty: int

ORDERS: dict[int, OrderOut] = {}

@app.post("/orders", response_model=OrderOut, status_code=status.HTTP_201_CREATED)
async def create_order(order: OrderIn) -> OrderOut:
    new = OrderOut(id=len(ORDERS) + 1, **order.model_dump())
    ORDERS[new.id] = new
    return new

@app.get("/orders/{order_id}", response_model=OrderOut)
async def get_order(order_id: int) -> OrderOut:
    if order_id not in ORDERS:
        raise HTTPException(status_code=404, detail="Order not found")
    return ORDERS[order_id]
```

What FastAPI does from the type hints:
- `order_id: int` in the path → parsed and validated (returns 422 if not an int).
- `order: OrderIn` (a Pydantic model) → read from the JSON body and validated.
- Plain `str`/`int` params not in the path → query parameters.
- `response_model` → output filtered and serialized, so you never leak extra fields (like `password_hash`).
- Auto docs at `/docs` (Swagger UI) and `/openapi.json`.

Run: `uvicorn main:app --reload` (dev) or `fastapi dev main.py`.

---

## 3. `async def` vs `def` (the most misunderstood part)

| You write | FastAPI does | Use when |
|---|---|---|
| `async def` | Runs on the event loop | You `await` async I/O (httpx, async DB driver, async LLM SDK) |
| `def` | Runs in a **threadpool** | You call blocking libraries (requests, sync DB driver, PIL) |

**The classic bug:** calling blocking code (`requests.get`, `time.sleep`, CPU-heavy NumPy/model inference) inside `async def`. It blocks the event loop and **every other request freezes**.

Fixes:
- Use async clients (`httpx.AsyncClient`, `AsyncOpenAI`, `asyncpg`, SQLAlchemy async).
- Or make the endpoint `def` (threadpool), or wrap: `await anyio.to_thread.run_sync(fn, arg)`.
- CPU-heavy work (your CV model inference!) → a separate worker/process or a model server (Triton, Ray Serve, TorchServe). Don't run it in the web process.

---

## 4. Dependency Injection (the FastAPI superpower)

A dependency is any callable whose result is injected into the endpoint. FastAPI resolves the graph, caches per-request, and runs cleanup.

```python
from typing import Annotated, AsyncIterator
from fastapi import Depends

async def get_db() -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as session:     # setup
        yield session                         # injected value
    # code after yield = teardown (runs after the response)

DB = Annotated[AsyncSession, Depends(get_db)]

def pagination(limit: int = 20, offset: int = 0) -> dict:
    return {"limit": min(limit, 100), "offset": offset}

@app.get("/items")
async def list_items(db: DB, page: Annotated[dict, Depends(pagination)]):
    ...
```

Used for: DB sessions, **current user / auth** ([07](07-fastapi-auth.md)), settings, rate limiting, feature flags.
For testing: `app.dependency_overrides[get_db] = fake_db`.

---

## 5. Application lifecycle: `lifespan`

Create expensive shared resources **once** (HTTP clients, DB pools, loaded ML models), not per request:

```python
from contextlib import asynccontextmanager
from openai import AsyncOpenAI

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.llm = AsyncOpenAI()
    app.state.model = load_vision_model()   # once at startup
    yield
    await app.state.llm.close()

app = FastAPI(lifespan=lifespan)
```

(`@app.on_event("startup")` is the old, deprecated way.)

---

## 6. Streaming LLM output (Server-Sent Events)

Users expect tokens to appear as they're generated.

```python
from fastapi import Request
from fastapi.responses import StreamingResponse

@app.post("/chat")
async def chat(body: ChatIn, request: Request):
    async def event_stream():
        stream = await request.app.state.llm.chat.completions.create(
            model=MODEL, messages=body.messages, stream=True,
        )
        async for chunk in stream:
            if await request.is_disconnected():
                break                              # stop paying for tokens nobody reads
            delta = chunk.choices[0].delta.content or ""
            if delta:
                yield f"data: {json.dumps({'delta': delta})}\n\n"
        yield "data: [DONE]\n\n"
    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

SSE (one-way, over plain HTTP, auto-reconnect) is the default for LLM streaming. Use **WebSockets** for real bidirectional needs (voice, collaborative editing, interrupting agents mid-run).

---

## 7. Other features you should know

| Feature | Notes |
|---|---|
| **Middleware** | CORS (`CORSMiddleware`), request IDs, timing, GZip. Runs on every request |
| **Exception handlers** | `@app.exception_handler(MyError)` → a consistent error envelope |
| **`APIRouter`** | Split routes per domain: `app.include_router(orders.router, prefix="/orders")` |
| **`BackgroundTasks`** | Fire-and-forget *after* the response, in-process. Fine for sending an email. **Not** for long or important jobs (lost on crash). Use a queue ([08](08-serving-llm-apps.md)) |
| **Settings** | `pydantic-settings` `BaseSettings` reads env vars and `.env` with validation |
| **File uploads** | `UploadFile` (needs `python-multipart`). Stream big files, check size and type |
| **`status` codes** | 200 OK, 201 Created, 204 No Content, 400 bad input, 401 not authenticated, 403 not allowed, 404, 409 conflict, 422 validation, 429 rate-limited, 5xx server |

---

## 8. Project layout (feature-based)

```
app/
  main.py               # create_app(), lifespan, include routers
  core/
    config.py           # Settings (pydantic-settings)
    security.py         # hashing, JWT helpers
    db.py               # engine, session dependency
  orders/               # one folder per domain
    router.py           # endpoints (thin)
    schemas.py          # Pydantic in/out models
    service.py          # business logic (no HTTP here)
    repository.py       # DB access
    models.py           # SQLAlchemy/SQLModel tables
  agents/
    router.py           # /chat, /runs
    loop.py             # the agent loop
    tools.py
tests/
alembic/                # DB migrations
```

Keep routers thin. Business logic in services can then be tested without HTTP, and reused from workers and agent tools.

---

## 9. Testing

```python
import pytest
from httpx import ASGITransport, AsyncClient
from app.main import app

@pytest.mark.anyio
async def test_create_order():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        r = await client.post("/orders", json={"item": "tea", "qty": 2})
    assert r.status_code == 201
    assert r.json()["qty"] == 2
```

Also: `fastapi.testclient.TestClient` for sync tests; `dependency_overrides` to swap DB and auth; mock the LLM client (never call real models in unit tests).

---

## 10. Deploying

- `uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers N` (or `fastapi run`), or Gunicorn managing Uvicorn workers.
- Docker image (slim base, `uv` for fast dependency installs), behind a reverse proxy or load balancer.
- Health endpoints (`/healthz` liveness, `/readyz` readiness).
- Remember: each worker is a separate process with its own memory, so in-memory dicts **are not shared**. Use Redis or Postgres for shared state.

---

## 11. Check yourself

- What happens if you call `requests.get()` inside an `async def` endpoint under load?
- How does `yield` in a dependency work, and when does the teardown run?
- Why use `response_model` even when you return the right object?
- Where should you load a 2 GB vision model in a FastAPI app? Where should you run inference?
- `BackgroundTasks` vs Celery/arq: when is each correct?
- How do you stop streaming (and paying for) tokens when the client disconnects?
