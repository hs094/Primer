# Adv 21 — Testing Dependencies with Overrides

🔑 `app.dependency_overrides[real_dep] = fake_dep` swaps a dependency for tests *without* touching production code. The whole DI graph re-resolves with the fake.

## Setup

```python
from fastapi import FastAPI, Depends

def get_settings():
    return {"env": "prod"}

app = FastAPI()

@app.get("/info")
async def info(s: dict = Depends(get_settings)):
    return s
```

## Override in test

```python
from fastapi.testclient import TestClient

def get_test_settings():
    return {"env": "test"}

app.dependency_overrides[get_settings] = get_test_settings

client = TestClient(app)
def test_info():
    assert client.get("/info").json() == {"env": "test"}
```

## Reset between tests

```python
import pytest
from fastapi.testclient import TestClient

@pytest.fixture
def client():
    app.dependency_overrides[get_settings] = get_test_settings
    yield TestClient(app)
    app.dependency_overrides.clear()
```

⚠️ Without `clear()`, overrides leak across tests and you get spooky failures.

## What gets overridden

The **callable** is the key. If your code has:

```python
async def get_db() -> AsyncIterator[AsyncSession]: ...
```

Override with another async-generator that yields a test session — DI resolves identically.

## Use cases

- Replace DB with an in-memory test DB or transactional rollback session.
- Inject a fake `current_user` instead of running JWT auth.
- Stub external HTTP clients with a recorded fixture.

💡 Overrides + `Depends()` everywhere is *the* reason FastAPI tests stay clean. If you find yourself monkeypatching, you're missing a dependency.

See [[FastAPI 24 - Testing and Debugging]] for the basics.
