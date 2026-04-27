# CO 13 — Strict Mode

🔑 **Key insight:** Strict mode flips Pydantic from "coerce when possible" to "reject anything that isn't already the right type" — and you can apply it at four granularities: per-call, per-field, per-type (via `Annotated`), or per-model.

## What strict disables

Default (lax) Pydantic accepts `'123'` for `int`, `1`/`0` for `bool`, ISO strings for `date`, etc. Under strict mode none of those coercions fire. Per the docs: "Pydantic will be much less lenient when coercing data, and will instead error if the data is not of the correct type." See the full lax→strict matrix in [[Pydantic CO 16 - Conversion Table]].

## Per-call: `strict=True` on `model_validate`

```python
from pydantic import BaseModel

class MyModel(BaseModel):
    x: int

MyModel.model_validate({'x': '123'})  # lax: converts to 123
MyModel.model_validate({'x': '123'}, strict=True)  # strict: raises ValidationError
```

Same flag works on [[Pydantic CO 14 - Type Adapter]]:

```python
TypeAdapter(date).validate_python('2000-01-01', strict=True)
```

## Per-field: `Field(strict=True)`

```python
from pydantic import BaseModel, Field

class User(BaseModel):
    name: str
    age: int = Field(strict=True)
```

Only `age` rejects coercion; `name` keeps default behavior.

## Per-type: `Annotated[..., Strict()]` and `StrictX` aliases

```python
from typing import Annotated
from pydantic import BaseModel, StrictInt, Strict
from uuid import UUID

class User(BaseModel):
    id: Annotated[UUID, Strict()]
    age: StrictInt  # Equivalent to Annotated[int, Strict()]
```

Convenience aliases: `StrictBool`, `StrictInt`, `StrictFloat`, `StrictStr`, `StrictBytes`. See [[Pydantic CO 05 - Types]].

## Per-model: `ConfigDict(strict=True)`

```python
from pydantic import BaseModel, ConfigDict, Field

class User(BaseModel):
    model_config = ConfigDict(strict=True)

    name: str
    age: int = Field(strict=False)  # Override: allows '18' conversion
```

Field-level `strict=False` overrides the model default. Mix-and-match is the common pattern: strict-by-default at the model, relaxed where you knowingly accept HTTP form input. See [[Pydantic CO 08 - Configuration]].

## JSON validation is laxer even under strict

⚠️ **Footgun:** strict mode is **looser when validating from JSON**. Date types accept ISO-string input via `model_validate_json` even with `strict=True`, because JSON has no native `date`/`datetime`/`UUID`/`Decimal`. If you want truly-only-the-Python-type, use `model_validate` on already-parsed Python data, not `model_validate_json`.

## When to reach for strict

- **Inter-service boundaries** where you control both ends and want type drift to fail loudly.
- **Database load paths** — you don't want `'true'` silently becoming `True`.
- **Test fixtures** — strict catches lazily-typed test data.

When **not** to: HTTP query/path parameters and form fields arrive as strings, period. Strict on those just forces you to coerce manually elsewhere — let Pydantic do it.

## Available strict aliases (cheat sheet)

```python
from pydantic import StrictBool, StrictInt, StrictFloat, StrictStr, StrictBytes
```

Each is `Annotated[T, Strict()]`. Use them directly as annotations:

```python
class User(BaseModel):
    name: StrictStr
    active: StrictBool
    score: StrictFloat
```

## Strict on `validate_call` and dataclasses

The `Strict()` annotation works inside [[Pydantic CO 15 - Validation Decorator]] argument types and [[Pydantic CO 11 - Dataclasses]] fields the same way as on `BaseModel` fields — it's a property of the type, not the model.

## Resolution order

When multiple strictness sources overlap:

1. Per-field `Field(strict=...)` wins over model config.
2. Per-call `strict=True/False` on `model_validate` wins over both.
3. `Annotated[T, Strict()]` is part of the type — overridden only by field-level or call-level explicit settings.

💡 **Takeaway:** Default Pydantic is forgiving; strict mode is the flag you set when you'd rather see a `ValidationError` than a quiet coercion that masks a real bug.
