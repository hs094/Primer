# Adv 20 — Testing Lifespan & Startup/Shutdown

🔑 `TestClient` only fires `lifespan` when used as a **context manager**. Outside the `with` block, no startup/shutdown hooks run.

## Lifespan recap

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.db = await connect()
    yield
    await app.state.db.close()

app = FastAPI(lifespan=lifespan)
```

See [[FastAPI 07 - Lifespan Events]].

## Test that triggers lifespan

```python
from fastapi.testclient import TestClient

def test_startup_runs():
    with TestClient(app) as client:          # ← context-manager form
        r = client.get("/items")
        assert r.status_code == 200
    # exit → shutdown ran here
```

⚠️ `client = TestClient(app)` followed by `client.get(...)` (no `with`) **skips** lifespan. State you initialised in startup will be missing.

## Asserting state

```python
def test_db_initialised():
    with TestClient(app) as client:
        assert app.state.db is not None
```

## Async tests

```python
import httpx, pytest

@pytest.mark.asyncio
async def test_async():
    async with app.router.lifespan_context(app):
        transport = httpx.ASGITransport(app=app)
        async with httpx.AsyncClient(transport=transport, base_url="http://test") as ac:
            r = await ac.get("/items")
            assert r.status_code == 200
```

Or pair `httpx.ASGITransport` with `LifespanManager` from `asgi-lifespan`.

## Why this exists

Real apps initialise expensive resources at startup (DB pools, ML models). Tests must exercise that path or you'll ship startup bugs.

🧪 Add at least one test that uses `with TestClient(app)` — it catches stupid lifespan regressions early.
