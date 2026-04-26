# Adv 25 — Advanced Python Types

🔑 Beyond the basics in [[Learn/Python Types Intro]] — `Annotated`, `Literal`, `TypedDict`, `NewType`, generics — and how FastAPI consumes them.

## `Annotated` — metadata on types

```python
from typing import Annotated
from fastapi import Query, Path, Depends

q: Annotated[str, Query(min_length=3, max_length=50, pattern="^foo")]
i: Annotated[int, Path(ge=1, le=1000)]
user: Annotated[User, Depends(get_current_user)]
```

Prefer `Annotated` over `Query(default)` defaults — keeps function defaults usable for actual defaults.

## `Literal` — enumerated values

```python
from typing import Literal

@app.get("/items/")
async def f(sort: Literal["asc", "desc"] = "asc"): ...
```

OpenAPI renders an enum. No need for `Enum` boilerplate for simple cases.

## Generics

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")

class Page(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int

@app.get("/users", response_model=Page[User])
async def list_users() -> Page[User]: ...
```

Pydantic v2 supports generic models — handy for paginated envelopes, RPC-style responses.

## `NewType`

```python
from typing import NewType
UserId = NewType("UserId", int)

def get(user_id: UserId): ...
```

Mypy treats `UserId` distinctly from `int` — no runtime cost. Useful for ID kind safety.

## `TypedDict` (less common in FastAPI)

```python
from typing import TypedDict

class Item(TypedDict):
    name: str
    price: float
```

Pydantic generally beats TypedDict for request/response — runtime validation. Use TypedDict only for internal-shape annotations.

## `Self` (PEP 673)

```python
from typing import Self

class Tree(BaseModel):
    children: list[Self] = []
```

Cleaner than forward-refs.

## `TYPE_CHECKING`

```python
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from .heavy_module import Heavy
```

Avoids import cycles & startup cost; types remain available to mypy.

💡 Type-driven design pays back in FastAPI: validators, docs, and the editor all key off these annotations.
