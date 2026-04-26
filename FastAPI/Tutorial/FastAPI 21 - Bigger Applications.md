# 21 — Bigger Applications (`APIRouter`)

🔑 Split your app into modules; each exports an `APIRouter`. Compose with `app.include_router(...)`.

## Structure

```
app/
├── __init__.py
├── main.py
├── dependencies.py
├── internal/
│   └── admin.py
└── routers/
    ├── items.py
    └── users.py
```

## A router module

```python
# app/routers/items.py
from fastapi import APIRouter, Depends, HTTPException
from app.dependencies import get_token_header

router = APIRouter(
    prefix="/items",
    tags=["items"],
    dependencies=[Depends(get_token_header)],
    responses={404: {"description": "Not found"}},
)

@router.get("/")
async def list_items(): ...

@router.get("/{id}")
async def get(id: int): ...
```

## Including in main app

```python
# app/main.py
from fastapi import Depends, FastAPI
from .dependencies import get_query_token
from .routers import items, users
from .internal import admin

app = FastAPI(dependencies=[Depends(get_query_token)])

app.include_router(users.router)
app.include_router(items.router)
app.include_router(
    admin.router,
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(get_token_header)],
    responses={418: {"description": "I'm a teapot"}},
)
```

- `include_router` can **override/extend** `prefix`, `tags`, `dependencies`, `responses` per include.
- Tags/deps stack: router's tags + include tags; router's deps + include deps.

## Nested routers

```python
parent = APIRouter(prefix="/v1")
parent.include_router(items.router)
app.include_router(parent)
```

## Reusing `Annotated` deps

Define `TokenDep = Annotated[str, Depends(get_token_header)]` once in `dependencies.py`; import everywhere.

## Gotchas

- ⚠️ `APIRouter` does not own lifespan or middleware. Attach these to the root `FastAPI` app.
- ⚠️ Route order matters across routers; FastAPI matches in inclusion order.
- 💡 Router-level `tags=[...]` groups endpoints in docs — treat tags as product-level features.
