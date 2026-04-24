Why you’d use **FastAPI’s `Depends` (dependency injection)** instead of just declaring and reusing a **global instance** (like a DB connection class, service class, etc.) at the top of your FastAPI app file.

Here’s a breakdown:

---

## 🔹 Global class declaration (naïve approach)

```python
# main.py
from fastapi import FastAPI
from db import Database

app = FastAPI()

db = Database()   # global instance

@app.get("/items")
def get_items():
    return db.fetch_items()
```

**Pros:**
- Very simple.
- Easy to see and use.
- Works fine for trivial apps.

**Cons:**
- **Hard to test**: You can’t easily swap out `db` for a mock/fake without monkey-patching.
- **Global state**: If your class holds state (like connections or config), it may leak across requests.
- **No per-request lifecycle**: Every request shares the same instance, even if you wanted a new instance per request.
- **Tight coupling**: All routes are tied to that single `db` variable.
---

## 🔹 Using `Depends` (dependency injection)

```python
from fastapi import Depends, FastAPI
from db import Database

app = FastAPI()

def get_db():
	db = Database()  # could also be a pooled connection
    try:
        yield db       # cleanup logic can go here
    finally:
        db.close()

@app.get("/items")
def get_items(db: Database = Depends(get_db)):
    return db.fetch_items()
```

**Pros:**
- **Lifecycle management**: FastAPI knows when to create and clean up dependencies (via `yield`).
- **Scoped control**: You can define per-request, per-session, or singleton lifetimes easily.
- **Testability**: During tests, you can override `get_db` with a mock:
```python
app.dependency_overrides[get_db] = lambda: FakeDB()
```
→ No monkey-patching needed.
- **Explicitness**: The function signature (`db: Database = Depends(...)`) makes dependencies very clear.
- **Flexibility**: Different routes can inject different variants (e.g., read-only DB, async client, external API client).
- **Asynchronous support**: Works cleanly with async context managers (`async with`).

**Cons:**
- Slightly more verbose to set up.
- Requires understanding dependency injection if you’re new to it.

---

## 🔹 When globals are okay

- Tiny scripts or prototypes.
- Services where lifecycle/scoping/testing don’t matter.
- Simple configs/constants (not services with state).

## 🔹 When `Depends` is better

- **Production apps** (where you want proper cleanup of DB connections, sessions, file handles).
- **Apps that will be tested** (easy overrides).
- **Apps with multiple environments/configs** (swap services in and out).
- **Anything async** (so dependencies don’t leak across requests).

---

✅ **Rule of thumb**:

- Globals = quick and dirty.
- `Depends` = scalable, testable, maintainable.

---