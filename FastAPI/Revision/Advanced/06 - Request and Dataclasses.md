# Adv 06 — Using `Request` Directly, Dataclasses, `Annotated` patterns

## Raw `Request` object

```python
from fastapi import Request

@app.get("/items/{id}")
def read(id: str, request: Request):
    return {
        "id": id,
        "client": request.client.host,
        "ua": request.headers.get("user-agent"),
        "scheme": request.url.scheme,
        "path_params": request.path_params,
        "query_params": dict(request.query_params),
    }
```

- Use when FastAPI's declarative params can't express what you need.
- `await request.body()` / `await request.json()` / `await request.form()` / `async for chunk in request.stream()` for raw access.
- `request.state` → place for middleware → handler state.

## Dataclasses as request / response models

```python
from dataclasses import dataclass
from pydantic.dataclasses import dataclass as pydantic_dataclass

@pydantic_dataclass
class Item:
    name: str
    price: float = 0.0

@app.post("/items/", response_model=Item)
async def create(item: Item) -> Item:
    return item
```

- Plain `@dataclass` works too, but Pydantic's version supports validation.
- For most cases, just use `BaseModel`; dataclasses help when integrating with existing code.

## `Annotated` patterns worth reusing

```python
from typing import Annotated
from fastapi import Depends, Query

# Reusable query-only param
Page = Annotated[int, Query(ge=1)]

# Reusable dependency alias
DBSession = Annotated[AsyncSession, Depends(get_session)]

# Combined validation + default
Limit = Annotated[int, Query(ge=1, le=100)]

@app.get("/")
async def f(page: Page = 1, limit: Limit = 10, session: DBSession):
    ...
```

- Cuts boilerplate across routers.
- Keep these in a `types.py` or `dependencies.py`.

## Gotchas

- ⚠️ Mixing `Annotated` with non-`Annotated` defaults in the same function signature is fine but less readable.
- ⚠️ Consuming `request.body()` / `request.form()` twice raises — read once and pass forward.
- ⚠️ Dataclasses with mutable defaults bite the usual Python way; Pydantic `Field(default_factory=...)` is safer.
