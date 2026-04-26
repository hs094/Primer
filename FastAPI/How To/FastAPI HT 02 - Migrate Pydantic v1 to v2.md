# HT 02 — Migrate Pydantic v1 → v2

🔑 FastAPI ≥ 0.100 supports Pydantic v2 natively. Migration is mechanical for most code; the ergonomic wins (speed, stricter validation) make it worth doing now.

## High-level steps

1. Bump deps: `pydantic = "^2"`, `fastapi >= 0.100`.
2. Run `bump-pydantic` codemod: `pip install bump-pydantic && bump-pydantic <pkg>`.
3. Fix what the codemod can't handle (see below).
4. Run tests; tighten validators.

## Common rewrites

```python
# v1
class Item(BaseModel):
    name: str
    class Config:
        orm_mode = True
        allow_population_by_field_name = True

# v2
class Item(BaseModel):
    name: str
    model_config = ConfigDict(from_attributes=True, populate_by_name=True)
```

```python
# v1: validators
@validator("name")
def upper(cls, v): return v.upper()

# v2
from pydantic import field_validator
@field_validator("name")
@classmethod
def upper(cls, v): return v.upper()
```

```python
# v1
item.dict(); item.json(); Item.parse_obj(d); Item.parse_raw(s)

# v2
item.model_dump(); item.model_dump_json(); Item.model_validate(d); Item.model_validate_json(s)
```

## Settings

```python
# v1
from pydantic import BaseSettings

# v2
from pydantic_settings import BaseSettings, SettingsConfigDict
```

`pydantic-settings` is a separate package now.

## Strictness changes

v2 is stricter about coercions: `"true"` → `bool` works, but `"1"` → `bool` raises by default unless you mark the field. Run your test suite end-to-end.

## ORM mode

`from_attributes=True` replaces `orm_mode=True` — needed when returning SQLAlchemy / ORM objects through `response_model=`.

## What you gain

- ~5–50× faster validation (Rust core).
- `Annotated[..., Field(...)]` patterns are first-class.
- Strict JSON schema generation matches OpenAPI 3.1.

⚠️ Custom JSON encoders (`Config.json_encoders`) are gone — use `@field_serializer` or `Annotated[T, PlainSerializer(...)]`.

💡 Migrate one module at a time, behind tests. Don't try a big-bang for a large codebase.
