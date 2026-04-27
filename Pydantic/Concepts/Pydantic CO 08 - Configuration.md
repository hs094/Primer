# CO 08 — Configuration

🔑 **Key insight:** All v2 model behaviour flows through `model_config = ConfigDict(...)` — the v1 inner `class Config` is deprecated, and configs *merge* down inheritance chains.

## The v2 form

```python
from pydantic import BaseModel, ConfigDict, ValidationError


class Model(BaseModel):
    model_config = ConfigDict(str_max_length=5)

    v: str


try:
    m = Model(v='abcdef')
except ValidationError as e:
    print(e)
    """
    1 validation error for Model
    v
      String should have at most 5 characters [type=string_too_long, input_value='abcdef', input_type=str]
    """
```

A plain dict (`{'str_max_length': 5}`) works too. **In Pydantic V1, the `Config` class was used. This is still supported, but deprecated.**

## Class-keyword form

Static type checkers see these. Limited to keys that are valid keyword arguments:

```python
from pydantic import BaseModel


class Model(BaseModel, frozen=True):
    a: str
```

## Knobs you actually reach for

| Key | Effect |
| --- | --- |
| `extra` | `'ignore'` (default) / `'allow'` / `'forbid'` — what to do with unknown fields. |
| `frozen` | Make instances immutable + hashable. |
| `populate_by_name` | Allow validating by Python field name *and* alias. |
| `arbitrary_types_allowed` | Permit non-Pydantic types as field annotations. |
| `str_strip_whitespace` | Strip whitespace from str inputs. |
| `str_to_lower` / `str_to_upper` | Case-coerce str inputs. |
| `str_max_length` / `str_min_length` | Global string length bounds. |
| `validate_assignment` | Re-validate on attribute assignment after init. |
| `validate_default` | Run validators on default values too. |
| `use_enum_values` | Store the `.value` instead of the Enum member. |
| `from_attributes` | ORM-style: read attrs from arbitrary objects (replaces v1 `orm_mode`). |
| `revalidate_instances` | `'never'` / `'always'` / `'subclass-instances'` — re-validate nested model instances. |
| `json_schema_extra` | Inject extra keys into the generated JSON Schema. See [[Pydantic CO 03 - JSON Schema]]. |
| `alias_generator` | Programmatic aliases. See [[Pydantic CO 07 - Alias]]. |
| `strict` | Strict-mode default for the whole model. See [[Pydantic CO 13 - Strict Mode]]. |

## Other carriers

Pydantic dataclasses — pass `config=` to the decorator (see [[Pydantic CO 11 - Dataclasses]]):

```python
from pydantic import ConfigDict
from pydantic.dataclasses import dataclass


@dataclass(config=ConfigDict(str_max_length=10, validate_assignment=True))
class User:
    name: str
```

`TypeAdapter` (see [[Pydantic CO 14 - Type Adapter]]):

```python
from pydantic import ConfigDict, TypeAdapter

ta = TypeAdapter(list[str], config=ConfigDict(coerce_numbers_to_str=True))
print(ta.validate_python([1, 2]))
#> ['1', '2']
```

Stdlib dataclasses — `__pydantic_config__` attribute. `TypedDict` — `@with_config(ConfigDict(...))`.

## Inheritance merges configs

A subclass inherits its parent's config and its own keys override on conflict:

```python
from pydantic import BaseModel, ConfigDict


class Parent(BaseModel):
    model_config = ConfigDict(extra='allow', str_to_lower=False)


class Model(Parent):
    model_config = ConfigDict(str_to_lower=True)

    x: str


m = Model(x='FOO', y='bar')
print(m.model_dump())
#> {'x': 'foo', 'y': 'bar'}
print(Model.model_config)
#> {'extra': 'allow', 'str_to_lower': True}
```

## Propagation into nested types

⚠️ Config does **not** automatically apply to nested Pydantic models — they keep their own config. But it *does* propagate into nested **stdlib** types that lack their own config:

```python
@dataclass
class UserWithoutConfig:
    name: str


@dataclass
@with_config(str_to_lower=False)
class UserWithConfig:
    name: str


class Parent(BaseModel):
    user_1: UserWithoutConfig
    user_2: UserWithConfig

    model_config = ConfigDict(str_to_lower=True)


print(Parent(user_1={'name': 'JOHN'}, user_2={'name': 'JOHN'}))
#> user_1=UserWithoutConfig(name='john') user_2=UserWithConfig(name='JOHN')
```

Centralise shared config on a `BaseModel` subclass — see [[Pydantic CO 01 - Models]].

💡 **Takeaway:** Pick your defaults once on a project-wide base class; rely on inheritance merging instead of repeating `ConfigDict` on every model.
