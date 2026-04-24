# 03 — Query Parameters

🔑 Any function argument **not** in the path template and **not** a Pydantic body model → query parameter.

## Basics

```python
@app.get("/items/")
async def list_items(skip: int = 0, limit: int = 10):
    ...
# GET /items/?skip=20&limit=50
```

- Default value → optional.
- No default → required (missing → 422).

## Optional

```python
q: str | None = None
```

## Bool coercion

`true, True, 1, yes, on` → `True`. `false, 0, no, off` → `False`.

## Multiple values (list)

```python
from typing import Annotated
from fastapi import Query

@app.get("/items/")
async def f(q: Annotated[list[str] | None, Query()] = None):
    ...
# ?q=a&q=b  → ["a", "b"]
```

- Without `Query(...)`, FastAPI thinks `list[str]` is a body.

## Required + optional mix

```python
async def f(needle: str, limit: int = 10, mode: str | None = None):
```

Order in source doesn't matter for request parsing, but helps humans.

## Query + path

```python
@app.get("/users/{user_id}/items/")
async def f(user_id: int, q: str | None = None):
```

Path param goes to `{user_id}`; any extra arg → query.

## Convert relative → absolute

Pydantic converts types; use `Query()` for validation rules (see [[05 - Query and Path Validation]]).

## Gotchas

- ⚠️ `bool = False` default makes the param optional with `False`; to distinguish "not sent" use `bool | None = None`.
- ⚠️ Repeated keys → list, single key → single value. Use `list[...]` for repeated.
