# 10 — Extra Data Types

🔑 Pydantic handles many types natively — just annotate and they're parsed, validated, documented.

## Common extras

| Type | JSON form | Notes |
|---|---|---|
| `datetime.datetime` | ISO 8601 string | tz-aware preferred |
| `datetime.date` | `"2025-01-30"` | |
| `datetime.time` | `"14:23:55"` | |
| `datetime.timedelta` | ISO 8601 duration or seconds | serialized as total seconds |
| `uuid.UUID` | string | |
| `bytes` | base64 string | |
| `Decimal` | number | |
| `frozenset[T]` | JSON array → set-like | unique |
| `Path` (`pathlib`) | string | |

## Example

```python
from datetime import datetime, timedelta
from uuid import UUID

@app.put("/items/{item_id}")
async def update(
    item_id: UUID,
    start: datetime,
    end: datetime,
    process_after: timedelta,
):
    duration = (end - start) + process_after
    return {"item_id": item_id, "duration": duration}
```

## Pydantic extras (explicit types)

- `HttpUrl`, `AnyUrl`, `PostgresDsn`, `RedisDsn`, etc.
- `EmailStr` (requires `email-validator`).
- `SecretStr` — hides value in `repr()`; access with `.get_secret_value()`.
- `IPvAnyAddress`, `IPv4Address`, `IPv6Address`.

## Constrained types (v2 `Annotated` pattern)

```python
from typing import Annotated
from pydantic import Field

PositiveInt = Annotated[int, Field(gt=0)]
UserName = Annotated[str, Field(min_length=3, max_length=30, pattern=r"^[a-z][a-z0-9_]*$")]
```

## Gotchas

- ⚠️ `bytes` body fields expect base64-encoded JSON strings. For raw bytes, use `File(...)`.
- ⚠️ `datetime` naive vs aware: pin to aware in the model where possible.
