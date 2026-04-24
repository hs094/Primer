# 17 — Dependency Injection with `Depends()`

🔑 A dependency is a callable. FastAPI resolves its parameters the same way as a path op (query/body/path/header/…) and injects the return value.

## Function dependency

```python
from fastapi import Depends
from typing import Annotated

async def pagination(skip: int = 0, limit: int = 100):
    return {"skip": skip, "limit": limit}

Pagination = Annotated[dict, Depends(pagination)]

@app.get("/items/")
async def list_items(p: Pagination):
    return p
```

`Annotated[..., Depends(x)]` is the idiomatic alias — reuse across routes.

## Class dependency

```python
class Paginator:
    def __init__(self, skip: int = 0, limit: int = 100):
        self.skip, self.limit = skip, limit

@app.get("/items/")
async def f(p: Annotated[Paginator, Depends()]):
    ...
```

`Depends()` (no arg) reuses the annotation class. Class params become query params.

## Sub-dependencies (nested chain)

```python
def a(token: str): ...
def b(x: Annotated[..., Depends(a)]): ...

@app.get("/")
def f(y: Annotated[..., Depends(b)]): ...
```

- Same dep appearing multiple times in a request is **cached** by default. Disable: `Depends(dep, use_cache=False)`.

## Path-op level dependencies (side effects only)

```python
def verify_token(x_token: Annotated[str, Header()]):
    if x_token != "secret":
        raise HTTPException(400, "X-Token header invalid")

@app.get("/items/", dependencies=[Depends(verify_token)])
async def f(): ...
```

Return value is discarded; use for auth / logging.

## Global dependencies

```python
app = FastAPI(dependencies=[Depends(verify_token)])
```

Or per-router: `APIRouter(dependencies=[...])`.

## Dependencies with `yield` (resource lifecycle)

```python
async def get_session() -> AsyncIterator[AsyncSession]:
    async with async_session_maker() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

- Code before `yield` = setup → runs per request.
- Yielded value is injected into handler.
- Code after `yield` = teardown → runs after response is sent (but before response finishes shipping, unless exception).
- Exceptions from the handler propagate into the `yield` block (after 0.106+, actually raised inside the generator).

## Dep return types

- Async or sync; FastAPI picks right strategy.
- Generator (`yield`) or context manager.
- Can `Depends` other things (chain).

## Gotchas

- ⚠️ Each identical `Depends` call in one request → cached. Fine for DB session; watch out if a dep captures request-time state you expect fresh.
- ⚠️ Teardown code runs **after** the response's body is produced; raising there won't change the HTTP status.
- 💡 Use `Depends(get_current_user)` for auth — see [[Tutorial/18 - Security]].
