# 05 — Query & Path Validation

🔑 Use `Query()` / `Path()` inside `Annotated[...]` to attach validation rules & metadata without changing the parameter's type.

## Preferred syntax

```python
from typing import Annotated
from fastapi import Query, Path

async def f(
    q: Annotated[str | None, Query(max_length=50)] = None,
    item_id: Annotated[int, Path(ge=1, le=1000, title="Item ID")] = ...,
):
```

`Annotated` keeps defaults in the normal Python place (right of `=`), decoupled from metadata.

## `Query()` options

| Option | Meaning |
|---|---|
| `default` | default value (positional or `default=`) |
| `alias` | external name differs from param name |
| `title`, `description` | docs metadata |
| `deprecated=True` | shown as deprecated in docs |
| `include_in_schema=False` | hide from OpenAPI |
| `min_length`, `max_length` | string length |
| `pattern` | regex (Pydantic v2 uses `pattern=`, not `regex=`) |
| `ge`, `le`, `gt`, `lt` | numeric bounds |
| `multiple_of` | number divisibility |
| `examples=[...]` | docs examples |

## Required with `...` vs no default

```python
q: Annotated[str, Query(min_length=3)]            # required
q: Annotated[str, Query(min_length=3)] = ...       # explicit Ellipsis = required
q: Annotated[str | None, Query()] = None           # optional
```

## Lists with validation

```python
q: Annotated[list[str], Query(min_length=1)] = []
```

## Path constraints

```python
item_id: Annotated[int, Path(ge=1, le=1000)]
```

- Path params are always required (they're in URL).
- `ge/le/gt/lt` and `multiple_of` same as `Query`.

## Alias for dashes / reserved names

```python
user_agent: Annotated[str | None, Query(alias="user-agent")] = None
```

## Old syntax (still works)

```python
async def f(q: str | None = Query(default=None, max_length=50)): ...
```

Prefer `Annotated` — cleaner and plays better with `Depends`.

## Gotchas

- ⚠️ `regex=` was renamed → use `pattern=` (Pydantic v2).
- ⚠️ Numeric 0 is falsy: `ge=0` allows 0; use `gt=0` for strictly positive.
