# 11b — Cookie & Header Parameter Models

🔑 Same idea as [[FastAPI 06 - Parameter Models]] for queries — bundle related cookies / headers into a Pydantic model so you can reuse + validate as a unit.

## Cookie model

```python
from typing import Annotated
from fastapi import Cookie, FastAPI
from pydantic import BaseModel

class Cookies(BaseModel):
    session_id: str
    fatebook_tracker: str | None = None

app = FastAPI()

@app.get("/items/")
async def f(cookies: Annotated[Cookies, Cookie()]):
    return cookies
```

Each model field reads its own cookie. Defaults / `None` make a cookie optional.

## Header model

```python
from fastapi import Header

class CommonHeaders(BaseModel):
    host: str
    x_token: list[str] = []
    user_agent: str | None = None

@app.get("/items/")
async def f(h: Annotated[CommonHeaders, Header()]): ...
```

- `_` ↔ `-` conversion still applies: `x_token` reads `X-Token`.
- Disable per-model: `Header(convert_underscores=False)`.

## Forbid extras

```python
class Cookies(BaseModel):
    model_config = {"extra": "forbid"}
    session_id: str
```

⚠️ Without this, unknown cookies/headers are silently dropped.

## Why models > flat params

- Reuse across routes (auth headers, tracking cookies).
- One docs entry, not N.
- Cross-field validators (`@field_validator`).

💡 Use models the moment two routes share the same set of cookies/headers.
