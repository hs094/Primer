# Adv 12 — Async Tests

🔑 `TestClient` is sync. For true async tests (e.g. async DB sessions inside the test), use `httpx.AsyncClient` with an ASGI transport.

## Install

```bash
pip install pytest pytest-asyncio httpx
```

## `pyproject.toml`

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"          # any async def test runs on the loop
```

## Async client fixture (FastAPI + httpx ASGI transport)

```python
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app

@pytest.fixture
async def client():
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as c:
        yield c
```

## An async test

```python
async def test_read_items(client):
    r = await client.get("/items/")
    assert r.status_code == 200
```

## Lifespan in tests

`ASGITransport` does **not** run lifespan by default. Options:

1. Drive lifespan manually:
   ```python
   async with LifespanManager(app):           # from asgi-lifespan
       async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
           yield c
   ```
2. Use `TestClient` when lifespan matters and the test body is sync.

## DB fixtures (async session per test)

```python
@pytest.fixture
async def db_session():
    async with AsyncSessionLocal() as s:
        yield s
        await s.rollback()            # keep tests isolated
```

Pair with `dependency_overrides[get_session] = lambda: db_session`.

## Parametrizing async tests

Same as usual: `@pytest.mark.parametrize` — works with `pytest-asyncio`.

## Gotchas

- ⚠️ Mixing `TestClient` (sync) and `AsyncClient` (async) in one test session is fine, but don't nest them.
- ⚠️ Forgetting `asyncio_mode = "auto"` → async tests silently skipped as "not run".
- ⚠️ `ASGITransport` bypasses the real network — CORS, HTTPS etc. aren't exercised. Integration tests can still be needed.
