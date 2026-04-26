# 24 — Testing & Debugging

## TestClient (sync, requests-like)

```python
# test_main.py
from fastapi.testclient import TestClient
from .main import app

client = TestClient(app)

def test_read_main():
    r = client.get("/")
    assert r.status_code == 200
    assert r.json() == {"msg": "Hello"}
```

- Under the hood uses `httpx`.
- Works for sync and async apps alike.

## Pytest fixtures

```python
import pytest

@pytest.fixture(scope="module")
def client():
    with TestClient(app) as c:     # "with" triggers lifespan events
        yield c

def test_create(client):
    r = client.post("/items/", json={"name": "x", "price": 1.0})
    assert r.status_code == 201
```

## Overriding dependencies (the killer feature)

```python
app.dependency_overrides[get_session] = lambda: fake_session
```

- Perfect for mocking auth, DB, external services.
- Remember to reset: `app.dependency_overrides = {}` between tests.

## Async tests

Use `pytest-asyncio` + `httpx.AsyncClient` — see [[Advanced/FastAPI 12 - Async Tests]].

## Testing WebSockets

```python
with client.websocket_connect("/ws") as ws:
    ws.send_text("hi")
    assert ws.receive_text() == "hi"
```

## Debugging

### Run from code (IDE breakpoints)

```python
# main.py
import uvicorn
if __name__ == "__main__":
    uvicorn.run("main:app", host="127.0.0.1", port=8000, reload=True)
```

Now you can launch `main.py` under a debugger.

### Validation errors

- Inspect `exc.errors()` in a `RequestValidationError` handler for debugging bad payloads.
- Set `logging.getLogger("fastapi").setLevel("DEBUG")` to trace more.

## Gotchas

- ⚠️ `TestClient` runs lifespan only if used as a context manager.
- ⚠️ `dependency_overrides` persists across tests in the same session — use a pytest fixture that resets.
- ⚠️ Pytest-asyncio modes: prefer `asyncio_mode = "auto"` in `pyproject.toml` / `pytest.ini`.
