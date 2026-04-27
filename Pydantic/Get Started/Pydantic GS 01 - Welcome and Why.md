# GS 01 — Welcome and Why

🔑 **Key insight:** Pydantic is the most widely used data validation library for Python — type-hint-driven schemas, Rust-backed validation, and a one-line bridge between untrusted input and typed Python objects.

## What it is

- Define a schema with Python 3.9+ type hints; validate untrusted data against it.
- Core validation lives in `pydantic-core`, written in Rust — among the fastest validation libraries for Python.
- Used by FastAPI, LangChain, Django Ninja, transformers, and ~8,000 other PyPI packages.
- Downloaded over 550M times per month; trusted by FAANG-scale teams in production.

## Hello world

```python
from datetime import datetime
from pydantic import BaseModel, PositiveInt

class User(BaseModel):
    id: int
    name: str = 'John Doe'
    signup_ts: datetime | None
    tastes: dict[str, PositiveInt]

external_data = {
    'id': 123,
    'signup_ts': '2019-06-01 12:22',
    'tastes': {'wine': 9, 'cheese': 7, 'cabbage': '1'},
}

user = User(**external_data)
```

The `'2019-06-01 12:22'` string is coerced to `datetime`; the `'1'` string under `cabbage` is coerced to a `PositiveInt`. Bad input raises a single rich `ValidationError` — see [[Pydantic ER 01 - Error Handling]].

## Why Pydantic — the highlights

### Type hints power the schema

The annotations *are* the schema. No DSL, no decorators per field. Plays well with mypy, PyCharm, Pyright, and your IDE's autocomplete out of the box. See [[Pydantic CO 01 - Models]].

### Performance

Validation logic is in Rust. The docs benchmark validating emoji URLs more than 3× faster than equivalent hand-written Python. Details and tuning in [[Pydantic CO 18 - Performance]].

### Serialization

Models round-trip to dicts, JSON-safe dicts, or JSON strings, with knobs to exclude unset fields, defaults, or `None` values. See [[Pydantic CO 09 - Serialization]].

### JSON Schema

> Pydantic is compliant with the latest version of JSON Schema specification (2020-12), which is compatible with OpenAPI 3.1.

This is what lets FastAPI generate OpenAPI docs for free.

### Strict mode and data coercion

Default is *lax* — sensible coercions like `'123'` → `123`. Strict mode disables coercion entirely; JSON validation sits in the middle (allows JSON-native conversions only). See [[Pydantic CO 13 - Strict Mode]].

### Four ways to validate

1. `BaseModel` — the rich, full-featured option ([[Pydantic CO 01 - Models]]).
2. Pydantic dataclasses — wraps stdlib `@dataclass` ([[Pydantic CO 11 - Dataclasses]]).
3. `TypeAdapter` — validate any standard type (`list[int]`, `TypedDict`, …) without a model ([[Pydantic CO 14 - Type Adapter]]).
4. `@validate_call` — validate function arguments and return values from a decorator.

### Customization

Custom validators and serializers per field or per model — including *wrap* validators that intercept the built-in pipeline. See [[Pydantic CO 10 - Validators]] and [[Pydantic CO 02 - Fields]].

### Ecosystem

> 466,400 repositories on GitHub and 8,119 packages on PyPI depend on Pydantic.

Including `transformers`, `LangChain`, and `FastAPI`. Settings management lives in a sibling package — see [[Pydantic CO 17 - Settings Management]].

### Battle-tested

550M+ monthly downloads, used at FAANG scale. Architecture details in [[Pydantic IT 01 - Architecture]].

## Where to go next

- New install? → [[Pydantic GS 03 - Installation]].
- Coming from V1? → [[Pydantic GS 04 - Migration Guide]].
- Stuck? → [[Pydantic GS 02 - Help with Pydantic]].
- Worried about upgrades? → [[Pydantic GS 05 - Version Policy]].

💡 **Takeaway:** If you already type-hint your code, you already know 80% of Pydantic — the rest is `BaseModel` and `model_validate`.
