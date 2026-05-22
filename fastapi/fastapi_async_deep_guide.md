# FastAPI Async Support — A Deep Guide

## Table of Contents

1. [How Python Handles Concurrency](#1-how-python-handles-concurrency)
2. [The Event Loop](#2-the-event-loop)
3. [Thread Pool & Executor](#3-thread-pool--executor)
4. [Coroutines, Tasks, and Awaitables](#4-coroutines-tasks-and-awaitables)
5. [The GIL and Why It Matters](#5-the-gil-and-why-it-matters)
6. [How FastAPI Implements Async Internally](#6-how-fastapi-implements-async-internally)
7. [Practical Examples Where Async Shines](#7-practical-examples-where-async-shines)
8. [Common Pitfalls and Anti-Patterns](#8-common-pitfalls-and-anti-patterns)
9. [Decision Guide — async def vs def](#9-decision-guide--async-def-vs-def)

---

## 1. How Python Handles Concurrency

Python offers three models of concurrency, each solving a different problem.

### 1.1 The Three Models

| Model | Best For | Python Tool |
|---|---|---|
| **Multiprocessing** | CPU-bound work (compute, ML, compression) | `multiprocessing`, `ProcessPoolExecutor` |
| **Threading** | Blocking I/O where you don't control the library | `threading`, `ThreadPoolExecutor` |
| **Async (cooperative)** | I/O-bound work at high concurrency | `asyncio`, `async/await` |

The key insight: **most web servers are I/O-bound, not CPU-bound.** A request handler typically spends 90%+ of its time waiting — for a database query, an HTTP call to another service, a file read. During that wait, the CPU sits idle. Async lets you use that idle time to handle other requests.

### 1.2 Concurrency vs. Parallelism

```
Concurrency: handling multiple things at once (interleaving)
Parallelism: doing multiple things at the same time (true simultaneous execution)

Async I/O  → concurrent, not parallel
Threading  → concurrent, usually not parallel (due to the GIL)
Multiproc  → concurrent AND parallel
```

Async achieves concurrency through **cooperative multitasking**: a task voluntarily yields control when it has nothing to do (e.g., waiting for a network response). The scheduler then runs another task. No OS thread switching, no locking overhead.

---

## 2. The Event Loop

The event loop is the engine at the heart of `asyncio`. Everything async runs inside it.

### 2.1 What It Is

The event loop is a **single-threaded scheduler** that:

1. Maintains a queue of ready-to-run coroutines (callbacks/tasks)
2. Monitors I/O file descriptors for readiness (via `select`/`epoll`/`kqueue`)
3. When a coroutine `await`s on I/O, it suspends the coroutine and registers the fd
4. When the I/O is ready, it marks the coroutine as runnable again
5. Runs coroutines one at a time — never two simultaneously

```
┌──────────────────────────────────────────────────────────┐
│                        Event Loop                        │
│                                                          │
│  ┌─────────┐    ┌──────────────┐    ┌─────────────────┐  │
│  │  Ready  │───▶│  Run Task A  │───▶│  Task A awaits  │  │
│  │  Queue  │    │  (coroutine) │    │   network I/O   │  │
│  └─────────┘    └──────────────┘    └────────┬────────┘  │
│       ▲                                       │          │
│       │          ┌──────────────┐             │ suspend  │
│       │          │  Run Task B  │◀────────────┘          │
│       │          │  (coroutine) │                        │
│       │          └──────┬───────┘                        │
│       │                 │ awaits db                      │
│       │                 │ suspend                        │
│       │          ┌──────▼───────┐                        │
│       │          │  I/O Poller  │  (epoll/kqueue)        │
│       │          │              │                        │
│       └──────────│  "A is ready"│                        │
│                  └──────────────┘                        │
└──────────────────────────────────────────────────────────┘
```

### 2.2 How `await` Works Under the Hood

When you write:

```python
result = await some_coroutine()
```

Python compiles this to a generator-like protocol. The coroutine is an object with a `send()` method. When it hits an `await`, it:

1. Yields a "Future" object upward to the event loop
2. The event loop registers the future's callback (e.g., "wake me when this socket is readable")
3. The event loop picks the next ready task from its queue and runs it
4. When the I/O completes, the callback fires, the future resolves, and the coroutine is rescheduled

This is why `await` can only be used inside `async def` — you can only yield from within a generator.

### 2.3 Running the Loop

```python
import asyncio

async def main():
    await asyncio.sleep(1)
    print("done")

# Python 3.7+
asyncio.run(main())  # creates a loop, runs until main() completes, closes loop
```

In a running process, there is typically **one event loop per thread** (though you can create more). FastAPI / Uvicorn creates the loop when the server starts.

---

## 3. Thread Pool & Executor

The event loop is single-threaded. But some I/O is inherently blocking (older DB drivers, file I/O, CPU work). For those, `asyncio` provides an **escape hatch into threads**.

### 3.1 `loop.run_in_executor`

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor(max_workers=10)

async def handler():
    loop = asyncio.get_event_loop()

    # Run blocking_function in a thread, await its completion
    result = await loop.run_in_executor(executor, blocking_function, arg1, arg2)
    return result
```

What happens:
- The blocking function runs in a **real OS thread** from the thread pool
- The event loop `await`s a `Future` that resolves when the thread finishes
- The event loop is **not blocked** — it can handle other coroutines during this time

### 3.2 `asyncio.to_thread` (Python 3.9+)

A cleaner wrapper:

```python
import asyncio

async def handler():
    result = await asyncio.to_thread(blocking_function, arg1, arg2)
    return result
```

### 3.3 Default Executor

`asyncio` has a default `ThreadPoolExecutor` used when you call `run_in_executor(None, ...)`. Its size is `min(32, os.cpu_count() + 4)`. FastAPI uses this to run regular `def` route handlers.

---

## 4. Coroutines, Tasks, and Awaitables

### 4.1 Coroutines

A coroutine is a function defined with `async def`. Calling it returns a **coroutine object** — it does not execute immediately.

```python
async def fetch(url: str) -> str:
    ...

coro = fetch("https://example.com")  # Nothing runs yet
result = await coro                   # Now it runs
```

### 4.2 Tasks

A `Task` wraps a coroutine and schedules it on the event loop immediately. Use tasks to run coroutines **concurrently** (not sequentially).

```python
import asyncio

async def main():
    # Sequential — total ~2 seconds
    await asyncio.sleep(1)
    await asyncio.sleep(1)

    # Concurrent — total ~1 second
    task1 = asyncio.create_task(asyncio.sleep(1))
    task2 = asyncio.create_task(asyncio.sleep(1))
    await task1
    await task2
```

### 4.3 `asyncio.gather`

Run multiple awaitables concurrently and collect results:

```python
results = await asyncio.gather(
    fetch_user(user_id),
    fetch_orders(user_id),
    fetch_recommendations(user_id),
)
user, orders, recs = results
```

All three requests fire simultaneously. Total time ≈ slowest single request.

### 4.4 Awaitables

Anything you can `await`:
- **Coroutines** — `async def` functions
- **Tasks** — `asyncio.create_task(...)`
- **Futures** — low-level future objects (usually from libraries)

---

## 5. The GIL and Why It Matters

The **Global Interpreter Lock (GIL)** is a mutex in CPython that allows only one thread to execute Python bytecode at a time.

### 5.1 Implications

```
Threading in Python:
  - Thread 1 runs → acquires GIL
  - Thread 2 wants to run → waits for GIL
  - Thread 1 does I/O → releases GIL
  - Thread 2 acquires GIL → runs

→ True parallelism only happens during I/O or C extension calls
→ CPU-bound threads don't get true parallelism
```

### 5.2 Why Async Sidesteps This

Async doesn't use multiple threads. It's **one thread, one GIL acquisition, many coroutines**. The GIL is never contended. Switching between coroutines is just a function call, not an OS thread context switch (~100x cheaper).

For CPU-bound work, use `ProcessPoolExecutor` — separate processes have separate GILs.

> **Python 3.13+** introduces an experimental "no-GIL" mode. This will eventually make threads viable for CPU-bound work, but async remains the best model for I/O-heavy web servers.

---

## 6. How FastAPI Implements Async Internally

### 6.1 The Stack

```
Your FastAPI app
       │
    FastAPI          ← routing, dependency injection, validation (Pydantic)
       │
    Starlette        ← ASGI framework, routing, middleware, WebSockets
       │
    Uvicorn          ← ASGI server, runs the event loop (via asyncio or uvloop)
       │
    asyncio / uvloop ← event loop implementation
       │
   OS (epoll/kqueue) ← kernel-level I/O notifications
```

FastAPI is built on **Starlette**, which is an ASGI framework. ASGI (Asynchronous Server Gateway Interface) is the async successor to WSGI.

### 6.2 ASGI — The Protocol

An ASGI app is a callable with this signature:

```python
async def app(scope, receive, send):
    ...
```

- `scope` — dict with request metadata (method, path, headers)
- `receive` — async callable to read the request body
- `send` — async callable to send response parts

Everything is async from the ground up. Uvicorn accepts a TCP connection, builds the scope, and calls your ASGI app as a coroutine.

### 6.3 How FastAPI Handles `async def` Routes

```python
@app.get("/users/{id}")
async def get_user(id: int, db: AsyncSession = Depends(get_db)):
    user = await db.get(User, id)
    return user
```

FastAPI:
1. Receives the ASGI call from Uvicorn
2. Matches the route
3. Resolves dependencies (also `await`s async dependencies)
4. **Directly `await`s** your route coroutine — it runs on the event loop thread
5. Serializes the response and `send`s it back

The route coroutine runs on the **main event loop thread**. Every `await` inside it yields control back to the loop, which can then handle other requests.

### 6.4 How FastAPI Handles Regular `def` Routes

```python
@app.get("/users/{id}")
def get_user(id: int):          # sync def, not async def
    user = db.query(User).get(id)
    return user
```

FastAPI cannot `await` a regular function. Instead, it uses `starlette.concurrency.run_in_threadpool`:

```python
# Starlette does this internally for sync route handlers
result = await run_in_threadpool(sync_route_function, *args, **kwargs)
```

Which is:
```python
async def run_in_threadpool(func, *args, **kwargs):
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, functools.partial(func, *args, **kwargs))
```

So your `def` route runs in the **default thread pool executor**, off the event loop thread. The event loop is freed to handle other requests while the thread runs.

```
Event Loop Thread             Thread Pool
──────────────────            ──────────────────────
Accept request A              
Call run_in_threadpool ──────▶ Thread 1: sync_handler(A)
[loop is free!]               │
Accept request B              │ (blocking DB call...)
Call run_in_threadpool ──────▶ Thread 2: sync_handler(B)
[loop is free!]               │
Accept request C              │
...                           │
Thread 1 finishes ◀───────────┘
Serialize & send A
```

### 6.5 Dependency Injection — Async All the Way Down

FastAPI's DI system fully supports async:

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session          # ← async context manager

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)   # async dep inside async dep
) -> User:
    user = await db.get(User, decode_token(token))
    return user

@app.get("/profile")
async def profile(user: User = Depends(get_current_user)):
    return user
```

FastAPI builds a dependency graph and resolves them concurrently where possible using `asyncio.gather` internally.

### 6.6 Uvloop — Drop-in Speed Boost

By default, asyncio uses Python's built-in event loop. Install `uvloop` (built on `libuv`, the same C library Node.js uses) for 2–4x faster event loop operations:

```bash
pip install uvloop
```

```python
# main.py
import uvloop
asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
```

Or just run via Uvicorn:
```bash
uvicorn main:app --loop uvloop
```

---

## 7. Practical Examples Where Async Shines

### 7.1 Parallel External API Calls

The canonical win for async: firing multiple HTTP requests simultaneously.

```python
import httpx
import asyncio
from fastapi import FastAPI

app = FastAPI()

@app.get("/dashboard/{user_id}")
async def dashboard(user_id: int):
    async with httpx.AsyncClient() as client:
        # Fire all three requests at the same time
        user_task    = asyncio.create_task(client.get(f"https://users-api/users/{user_id}"))
        orders_task  = asyncio.create_task(client.get(f"https://orders-api/users/{user_id}"))
        recs_task    = asyncio.create_task(client.get(f"https://recs-api/users/{user_id}"))

        user_resp, orders_resp, recs_resp = await asyncio.gather(
            user_task, orders_task, recs_task
        )

    return {
        "user":   user_resp.json(),
        "orders": orders_resp.json(),
        "recs":   recs_resp.json(),
    }

# Sequential: ~300ms (100 + 100 + 100ms)
# Concurrent: ~100ms (all in parallel)
```

### 7.2 Async Database Access

Using `asyncpg` / `SQLAlchemy async` / `databases` library:

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
from fastapi import FastAPI, Depends

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(User).where(User.id == user_id)
    )
    user = result.scalar_one_or_none()
    if not user:
        raise HTTPException(status_code=404)
    return user
```

Each DB query yields the event loop while waiting for Postgres to respond. Under load, many requests can have pending DB queries simultaneously — all interleaved on one thread.

### 7.3 Async Redis / Cache

```python
import redis.asyncio as aioredis
from fastapi import FastAPI

app = FastAPI()
redis = aioredis.from_url("redis://localhost")

@app.get("/product/{product_id}")
async def get_product(product_id: int, db: AsyncSession = Depends(get_db)):
    # Check cache first
    cached = await redis.get(f"product:{product_id}")
    if cached:
        return json.loads(cached)

    # Cache miss — fetch from DB
    product = await db.get(Product, product_id)
    if not product:
        raise HTTPException(status_code=404)

    # Store in cache (non-blocking)
    await redis.setex(f"product:{product_id}", 300, product.json())
    return product
```

### 7.4 WebSockets

Async is essentially mandatory for WebSockets — you need to receive messages and send broadcasts simultaneously.

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        self.active: List[WebSocket] = []

    async def connect(self, ws: WebSocket):
        await ws.accept()
        self.active.append(ws)

    def disconnect(self, ws: WebSocket):
        self.active.remove(ws)

    async def broadcast(self, message: str):
        # Send to all connected clients concurrently
        await asyncio.gather(*[ws.send_text(message) for ws in self.active])

manager = ConnectionManager()

@app.websocket("/ws/{room_id}")
async def websocket_endpoint(ws: WebSocket, room_id: str, user: str):
    await manager.connect(ws)
    try:
        while True:
            data = await ws.receive_text()          # yields event loop while waiting
            await manager.broadcast(f"{user}: {data}")
    except WebSocketDisconnect:
        manager.disconnect(ws)
        await manager.broadcast(f"{user} left the room")
```

### 7.5 Background Tasks

FastAPI's `BackgroundTasks` runs work after the response is sent, on the event loop:

```python
from fastapi import FastAPI, BackgroundTasks
import httpx

app = FastAPI()

async def send_welcome_email(email: str):
    async with httpx.AsyncClient() as client:
        await client.post("https://email-service/send", json={
            "to": email,
            "subject": "Welcome!",
            "body": "Thanks for signing up."
        })

async def log_signup(user_id: int):
    await redis.incr("signups:total")
    await redis.lpush("signups:recent", user_id)

@app.post("/register")
async def register(user_data: UserCreate, bg: BackgroundTasks, db: AsyncSession = Depends(get_db)):
    user = User(**user_data.dict())
    db.add(user)
    await db.commit()
    await db.refresh(user)

    # These run after response is sent — non-blocking for the client
    bg.add_task(send_welcome_email, user.email)
    bg.add_task(log_signup, user.id)

    return {"id": user.id, "status": "created"}
```

### 7.6 Streaming Responses

Async generators let you stream large responses without buffering everything in memory:

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import httpx

app = FastAPI()

@app.get("/stream/large-report")
async def stream_report(db: AsyncSession = Depends(get_db)):
    async def generate():
        yield '{"rows": [\n'
        first = True
        # Stream rows from DB cursor
        async for row in await db.stream(select(Report).order_by(Report.created_at)):
            if not first:
                yield ",\n"
            yield json.dumps(row._mapping)
            first = False
        yield "\n]}"

    return StreamingResponse(generate(), media_type="application/json")

@app.get("/proxy/upstream")
async def proxy_upstream():
    """Proxy a large response from upstream without buffering"""
    async def generate():
        async with httpx.AsyncClient() as client:
            async with client.stream("GET", "https://upstream/big-file") as response:
                async for chunk in response.aiter_bytes(chunk_size=8192):
                    yield chunk

    return StreamingResponse(generate(), media_type="application/octet-stream")
```

### 7.7 Startup / Shutdown Lifespan Events

Async lifespan lets you initialize connection pools before the first request:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    app.state.redis = await aioredis.from_url("redis://localhost")
    app.state.http_client = httpx.AsyncClient()
    engine = create_async_engine(DATABASE_URL)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    app.state.engine = engine
    print("Startup complete")

    yield  # App is running

    # Shutdown
    await app.state.redis.close()
    await app.state.http_client.aclose()
    await app.state.engine.dispose()
    print("Shutdown complete")

app = FastAPI(lifespan=lifespan)

@app.get("/ping")
async def ping(request: Request):
    pong = await request.app.state.redis.ping()
    return {"redis": pong}
```

### 7.8 Async Middleware

```python
import time
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()
        response = await call_next(request)     # ← await the rest of the chain
        duration = time.perf_counter() - start
        response.headers["X-Process-Time"] = f"{duration:.4f}"
        return response

app.add_middleware(TimingMiddleware)
```

### 7.9 Mixing Sync and Async — The Right Way

Sometimes you must call a sync library (e.g., legacy ORM, CPU-bound task):

```python
import asyncio
from fastapi import FastAPI

app = FastAPI()

def cpu_heavy_computation(data: list) -> dict:
    """This blocks — cannot await it directly"""
    return {"result": sum(x ** 2 for x in data)}   # toy example

def legacy_sync_db_call(user_id: int) -> dict:
    """Old psycopg2 / synchronous SQLAlchemy"""
    with SyncSessionLocal() as session:
        return session.query(User).get(user_id).__dict__

@app.get("/compute/{user_id}")
async def compute(user_id: int):
    loop = asyncio.get_event_loop()

    # Run both blocking calls in thread pool — concurrently
    user_task = loop.run_in_executor(None, legacy_sync_db_call, user_id)
    compute_task = loop.run_in_executor(None, cpu_heavy_computation, list(range(10000)))

    user, computed = await asyncio.gather(user_task, compute_task)
    return {"user": user, "computed": computed}
```

---

## 8. Common Pitfalls and Anti-Patterns

### 8.1 Calling Blocking Code in an `async def` Route

```python
# ❌ BAD — blocks the entire event loop
@app.get("/users/{id}")
async def get_user(id: int):
    time.sleep(1)                         # blocks event loop!
    user = requests.get(f"...")           # blocking HTTP — freezes all other requests
    return user.json()

# ✅ GOOD — use async equivalents
@app.get("/users/{id}")
async def get_user(id: int):
    await asyncio.sleep(1)                # yields event loop
    async with httpx.AsyncClient() as c:
        resp = await c.get(f"...")        # async HTTP
    return resp.json()
```

### 8.2 Using a Sync ORM in an Async Route Without `run_in_executor`

```python
# ❌ BAD
@app.get("/users/{id}")
async def get_user(id: int, db = Depends(get_sync_db)):
    return db.query(User).get(id)   # synchronous SQLAlchemy — blocks event loop

# ✅ OPTION A — switch to async SQLAlchemy
@app.get("/users/{id}")
async def get_user(id: int, db: AsyncSession = Depends(get_async_db)):
    return await db.get(User, id)

# ✅ OPTION B — use regular def (FastAPI will run it in thread pool)
@app.get("/users/{id}")
def get_user(id: int, db = Depends(get_sync_db)):
    return db.query(User).get(id)
```

### 8.3 Creating Tasks Without Awaiting or Storing Them

```python
# ❌ BAD — fire-and-forget Task gets garbage collected
@app.post("/notify")
async def notify(user_id: int):
    asyncio.create_task(send_notification(user_id))   # may be GC'd before it runs!
    return {"status": "ok"}

# ✅ GOOD — use BackgroundTasks for fire-and-forget
@app.post("/notify")
async def notify(user_id: int, bg: BackgroundTasks):
    bg.add_task(send_notification, user_id)
    return {"status": "ok"}
```

### 8.4 Not Using Connection Pools

```python
# ❌ BAD — new client per request is expensive
@app.get("/data")
async def get_data():
    async with httpx.AsyncClient() as client:   # creates TCP connection each time
        return await client.get("https://api/data")

# ✅ GOOD — shared client with connection pool
http_client: httpx.AsyncClient = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global http_client
    http_client = httpx.AsyncClient(limits=httpx.Limits(max_connections=100))
    yield
    await http_client.aclose()

@app.get("/data")
async def get_data():
    return await http_client.get("https://api/data")
```

### 8.5 Async in Tests

```python
# Use pytest-asyncio
import pytest
from httpx import AsyncClient, ASGITransport

@pytest.mark.asyncio
async def test_get_user():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        response = await client.get("/users/1")
    assert response.status_code == 200
```

---

## 9. Decision Guide — `async def` vs `def`

Use this flowchart when writing a route or dependency:

```
Does this function do any I/O?
│
├── YES
│    │
│    ├── Is there an async-native library for it?
│    │    ├── YES → async def  +  await the library calls
│    │    │         (asyncpg, httpx, aioredis, motor, aioboto3 ...)
│    │    │
│    │    └── NO  → def  (FastAPI runs it in thread pool automatically)
│    │              OR: wrap with asyncio.to_thread() / run_in_executor()
│    │
│    └── Is it CPU-bound AND expensive (>5ms)?
│         ├── YES → ProcessPoolExecutor  +  loop.run_in_executor
│         └── NO  → async def is fine; brief CPU work doesn't hurt
│
└── NO (pure computation, no I/O)
     └── def  (no benefit to async; avoid adding overhead)
```

### Summary Table

| Scenario | Recommendation |
|---|---|
| Async DB driver (asyncpg, async SQLAlchemy) | `async def` |
| Sync DB driver (psycopg2, sync SQLAlchemy) | `def` (thread pool) |
| HTTP calls | `async def` + `httpx.AsyncClient` |
| Redis | `async def` + `redis.asyncio` |
| File I/O (large files) | `def` or `asyncio.to_thread` |
| CPU computation | `def` + `ProcessPoolExecutor` |
| WebSockets | `async def` (required) |
| Quick computation (no I/O) | `def` |

---

## Further Reading

- [Python `asyncio` docs](https://docs.python.org/3/library/asyncio.html)
- [FastAPI async docs](https://fastapi.tiangolo.com/async/)
- [Starlette source — `concurrency.py`](https://github.com/encode/starlette/blob/master/starlette/concurrency.py)
- [ASGI spec](https://asgi.readthedocs.io/en/latest/)
- [uvloop](https://github.com/MagicStack/uvloop)
- [SQLAlchemy async engine](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
