# 11 — Cookies & Headers

🔑 Declare with `Cookie(...)` / `Header(...)` in `Annotated`. Same validators as `Query()`.

## Single cookie

```python
from fastapi import Cookie
from typing import Annotated

@app.get("/items/")
async def f(ads_id: Annotated[str | None, Cookie()] = None):
    ...
```

## Single header

```python
from fastapi import Header

@app.get("/items/")
async def f(user_agent: Annotated[str | None, Header()] = None):
    ...
```

- `_` ↔ `-` conversion: `user_agent` reads `User-Agent`.
- Disable: `Header(convert_underscores=False)`.

## Duplicate headers → list

```python
x_token: Annotated[list[str] | None, Header()] = None
# curl -H "x-token: a" -H "x-token: b"  → ["a", "b"]
```

## Cookie / header as models

See [[FastAPI 06 - Parameter Models]] — `Annotated[Model, Cookie()]` or `Annotated[Model, Header()]` groups them.

## Setting response cookies / headers

Via response mutator:

```python
from fastapi import Response

@app.get("/")
def f(response: Response):
    response.set_cookie(key="session", value="xyz", httponly=True)
    response.headers["X-Custom"] = "1"
    return {"ok": True}
```

Or construct a custom `Response` / `JSONResponse` directly. See [[Advanced/FastAPI 03 - Additional Responses]].

## Gotchas

- ⚠️ Header names are case-insensitive; FastAPI normalizes for you.
- ⚠️ `Cookie()` ≠ `Set-Cookie`. To *set* cookies, use the `Response` object, not a parameter.
