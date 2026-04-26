# Python Types Intro

🔑 FastAPI uses standard Python type hints as the *single* contract: validation, serialization, docs, editor autocomplete — all driven by your annotations.

## Basic types

```python
def greet(name: str, age: int, active: bool = True) -> str:
    return f"{name}/{age}/{active}"
```

## Built-in generics (3.9+) — preferred

```python
items: list[str]
mapping: dict[str, int]
pair: tuple[int, str]
unique: set[bytes]
```

⚠️ Don't import `List`/`Dict`/`Tuple`/`Set` from `typing` — they're legacy.

## Union & optional

```python
x: int | str          # PEP 604 union
y: int | None = None  # replaces Optional[int]
```

## `Annotated` — metadata travels with the type

```python
from typing import Annotated
from fastapi import Query

q: Annotated[str | None, Query(max_length=50)] = None
```

💡 `Annotated[T, *meta]` lets FastAPI attach `Query()`, `Path()`, `Body()`, `Depends()` to the type without overloading the default value. Prefer `Annotated` everywhere.

## Pydantic models as types

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    tags: list[str] = []

def create(item: Item): ...
```

A Pydantic-typed parameter ⇒ JSON body parsed, validated, and documented for free.

## Why this matters in FastAPI

- Path / query / header / cookie / body — all resolved from type hints.
- Response shape resolved from return annotation (or `response_model=`).
- Editor + `mypy` agree with runtime behavior.

🧪 If you can express a constraint in a type, do it there — never re-validate at the top of a handler.
