# 06 — Parameter Models (Query / Header / Cookie)

🔑 Group related **non-body** params into a single Pydantic model. Reuse, compose, and validate them together.

## Query params as a model

```python
from typing import Annotated, Literal
from fastapi import Query
from pydantic import BaseModel, Field

class FilterParams(BaseModel):
    model_config = {"extra": "forbid"}         # 422 unknown query keys
    limit: int = Field(100, gt=0, le=100)
    offset: int = Field(0, ge=0)
    order_by: Literal["created_at", "updated_at"] = "created_at"
    tags: list[str] = []

@app.get("/items/")
async def list_items(filter_q: Annotated[FilterParams, Query()]):
    ...
```

- Requests still use flat query keys: `?limit=10&offset=20&tags=a&tags=b`.
- Model fields appear individually in OpenAPI.

## Header params as a model

```python
from fastapi import Header

class CommonHeaders(BaseModel):
    host: str
    save_data: bool = False
    if_modified_since: str | None = None
    x_tag: list[str] = []

@app.get("/items/")
async def f(headers: Annotated[CommonHeaders, Header()]):
    ...
```

- Underscores in Python → hyphens in HTTP (`x_tag` → `x-tag`). Override with `Header(convert_underscores=False)` or `alias`.

## Cookie params as a model

```python
from fastapi import Cookie

class Cookies(BaseModel):
    session_id: str
    fatebook_tracker: str | None = None

@app.get("/items/")
async def f(cookies: Annotated[Cookies, Cookie()]):
    ...
```

## Forbid extras

Set `model_config = {"extra": "forbid"}` to reject unexpected headers / cookies / query keys.

## Gotchas

- ⚠️ Each field becomes a **separate** parameter in docs; the model is just an organizer.
- ⚠️ Don't use `Body()` here — these are not body parameters.
- 💡 Use param models to reuse common filters across many routes.
