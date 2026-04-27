# GS 04 — Migration Guide (V1 → V2)

🔑 **Key insight:** Almost every public method on `BaseModel` was renamed with a `model_` prefix; `Config` became `model_config`; validators got a `field_` / `model_` split — `bump-pydantic` does most of the typing for you.

## Install V2 + automated codemod

```bash
pip install -U pydantic
pip install bump-pydantic
bump-pydantic my_package
```

Run on a clean git tree, review the diff, then migrate by hand for what it misses. V1 stays accessible inside V2 under `pydantic.v1` (works in V1 1.10.17+ and V2), so you can migrate module-by-module:

```python
from pydantic.v1 import BaseModel
```

## BaseModel method renames

| V1 | V2 |
|---|---|
| `__fields__` | `model_fields` |
| `dict()` | `model_dump()` |
| `json()` | `model_dump_json()` |
| `parse_obj(obj)` | `model_validate(obj)` |
| `parse_raw(s)` | `model_validate_json(s)` *(or deprecated)* |
| `parse_file(p)` | (removed — load + `model_validate_json`) |
| `from_orm(obj)` | `model_validate(obj)` with `model_config = ConfigDict(from_attributes=True)` |
| `construct(...)` | `model_construct(...)` |
| `copy()` | `model_copy()` |
| `schema()` | `model_json_schema()` |
| `schema_json()` | `json.dumps(model_json_schema())` |
| `update_forward_refs()` | `model_rebuild()` |

⚠️ Equality is stricter in V2: two models are equal only when both are `BaseModel` instances of the *same* type with identical field values, extras, and private attributes.

## `GenericModel` is gone

Replace `pydantic.generics.GenericModel` with plain `BaseModel` + `typing.Generic`:

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar('T')

class Container(BaseModel, Generic[T]):
    item: T
```

⚠️ Don't `isinstance(x, Container[int])` — parametrized generics can't be used in `isinstance` checks. Subclass for a concrete type.

## Config: `class Config` → `model_config`

```python
from pydantic import BaseModel, ConfigDict

class User(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,
        populate_by_name=True,
    )
    name: str
```

### Renamed config keys

| V1 | V2 |
|---|---|
| `allow_population_by_field_name` | `populate_by_name` |
| `orm_mode` | `from_attributes` |
| `anystr_lower` | `str_to_lower` |
| `anystr_strip_whitespace` | `str_strip_whitespace` |
| `anystr_upper` | `str_to_upper` |
| `min_anystr_length` | `str_min_length` |
| `max_anystr_length` | `str_max_length` |
| `schema_extra` | `json_schema_extra` |

### Removed config keys

`allow_mutation`, `error_msg_templates`, `fields`, `getter_dict`, `json_loads`, `json_dumps`, `json_encoders` (use a custom serializer instead), `post_init_call`, `smart_union`.

Full list and replacements in [[Pydantic CO 08 - Configuration]].

## Validators: split into `field_validator` and `model_validator`

```python
from pydantic import BaseModel, field_validator, model_validator

class Item(BaseModel):
    name: str

    @field_validator('name')
    @classmethod
    def upper(cls, v: str) -> str:
        return v.upper()

    @model_validator(mode='after')
    def check(self) -> 'Item':
        ...
        return self
```

Key V2 changes:

- `each_item=True` is gone — put the validator on the inner type via `Annotated[...]`.
- Drop the `field` and `config` arguments from validator signatures.
- `@model_validator(mode='before')` receives the raw dict; `mode='after'` receives the constructed instance (via `self`).
- `TypeError` raised inside a validator no longer auto-converts to `ValidationError` — raise `ValueError` (or `AssertionError`) instead.
- `allow_reuse=True` removed; reuse helpers via `Annotated` with `AfterValidator` / `BeforeValidator`.
- `@validate_arguments` → `@validate_call`.

Deeper dive in [[Pydantic CO 10 - Validators]].

## Custom types

| V1 | V2 |
|---|---|
| `__get_validators__` | `__get_pydantic_core_schema__` |
| `__modify_schema__` | `__get_pydantic_json_schema__` |

V2's recommended pattern is `typing.Annotated[T, ...]` with `BeforeValidator`/`AfterValidator`/`PlainSerializer` rather than subclassing — see [[Pydantic CO 05 - Types]].

## `TypeAdapter` replaces `parse_obj_as` / `schema_of`

```python
from pydantic import TypeAdapter

adapter = TypeAdapter(list[int])
assert adapter.validate_python(['1', '2', '3']) == [1, 2, 3]
adapter.json_schema()
```

See [[Pydantic CO 14 - Type Adapter]].

## `RootModel` replaces `__root__`

V1 used `__root__: list[int]` inside a `BaseModel`. V2:

```python
from pydantic import RootModel
IntList = RootModel[list[int]]
IntList([1, 2, 3]).root  # [1, 2, 3]
```

## Type-handling changes

- **Unions** — V2 picks the best match, not left-to-right. Add `Field(union_mode='left_to_right')` for V1 behavior.
- **`Optional[T]`** — *required, may be `None`*. Not auto-defaulted to `None`; add `= None` explicitly.
- **`int` from `float`** — only when the decimal part is zero (`3.0` → `3`, `3.5` raises).
- **Regex** — Rust `regex` crate, no lookarounds/backreferences. `regex_engine='python-re'` config to fall back.
- **Dicts** — iterables of pairs no longer coerce to `dict`.

## Dataclasses, moved types, removed features

Dataclass changes ([[Pydantic CO 11 - Dataclasses]]):

- `__post_init__` runs *after* validation, not before.
- No tuple → dict coercion; `extra='allow'` unsupported (use `'ignore'`).
- No `__pydantic_model__`; wrap with `TypeAdapter` for schema access.

Moved out of the main package:

- `BaseSettings` → `from pydantic_settings import BaseSettings` ([[Pydantic CO 17 - Settings Management]]).
- `Color`, `PaymentCardNumber` → `pydantic-extra-types`.
- URL types no longer subclass `str` — use `str(url)`.

`Constrained*` classes are gone — use `Annotated`:

```python
from typing import Annotated
from pydantic import Field
PositiveSmallInt = Annotated[int, Field(ge=0, lt=1000)]
```

Error subclasses collapsed into `ValidationError` with structured `errors()` output — see [[Pydantic ER 01 - Error Handling]].

## Mypy + strict mode notes

Run V1 and V2 plugins side-by-side during migration:

```toml
[tool.mypy]
plugins = ["pydantic.mypy", "pydantic.v1.mypy"]
```

Strict-by-default kicks in for `model_validate_json` and JSON-mode parsing of some types. Lax coercion still rules `model_validate` on plain Python input — see [[Pydantic CO 13 - Strict Mode]].

💡 **Takeaway:** Run `bump-pydantic`, swap `Config` for `model_config = ConfigDict(...)`, split your `@validator`s into `@field_validator` / `@model_validator`, then chase the `Optional`/union/regex edge cases.
