# CO 07 — Alias

🔑 **Key insight:** `alias` is the symmetric shortcut; `validation_alias` and `serialization_alias` decouple the two directions when wire formats and Python names diverge.

## The three field-level aliases

```python
from pydantic import BaseModel, Field


class Model(BaseModel):
    my_field: str = Field(alias='my_alias')
```

```python
class Model(BaseModel):
    my_field: str = Field(validation_alias='my_alias')
```

```python
class Model(BaseModel):
    my_field: str = Field(serialization_alias='my_alias')
```

- `alias` — used for both directions.
- `validation_alias` — input only. Accepts `str`, `AliasPath`, or `AliasChoices`.
- `serialization_alias` — output only.

## AliasPath: pull from nested input

```python
from pydantic import BaseModel, Field, AliasPath


class User(BaseModel):
    first_name: str = Field(validation_alias=AliasPath('names', 0))
    address: str = Field(validation_alias=AliasPath('contact', 'address'))


user = User.model_validate({
    'names': ['John', 'Doe'],
    'contact': {'address': '221B Baker Street'}
})
```

Path elements can be dict keys (str) or list indices (int).

## AliasChoices: try several keys

Earlier choices win:

```python
from pydantic import BaseModel, Field, AliasChoices


class User(BaseModel):
    first_name: str = Field(validation_alias=AliasChoices('first_name', 'fname'))
```

Mix `AliasChoices` with `AliasPath`:

```python
class User(BaseModel):
    first_name: str = Field(
        validation_alias=AliasChoices('first_name', AliasPath('names', 0))
    )
```

## Alias generators

Apply a transform across every field — bridge `snake_case` Python to `camelCase` JSON without touching each `Field`:

```python
from pydantic import BaseModel, ConfigDict


class Tree(BaseModel):
    model_config = ConfigDict(
        alias_generator=lambda field_name: field_name.upper()
    )
    age: int
    height: float


t = Tree.model_validate({'AGE': 12, 'HEIGHT': 1.2})
print(t.model_dump(by_alias=True))  # {'AGE': 12, 'HEIGHT': 1.2}
```

Built-in generators: `to_pascal`, `to_camel`, `to_snake`. Use `AliasGenerator(validation_alias=..., serialization_alias=...)` to split the two directions.

## Precedence

Explicit `Field(alias=...)` overrides the generator by default. Tune via `alias_priority`:

- `alias_priority=2` — explicit alias *wins*, generator does **not** override it.
- `alias_priority=1` — explicit alias gets *overridden* by the generator.
- Unset — Pydantic's default behavior.

```python
class Voice(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel)
    language_code: str = Field(alias='lang')
```

## Validation and serialization toggles

```python
class Model(BaseModel):
    my_field: str = Field(alias='my_alias')
    model_config = ConfigDict(
        validate_by_alias=True,    # default
        validate_by_name=False,    # default
        serialize_by_alias=True,   # opt-in
    )
```

Per-call override: `Model.model_validate(data, by_alias=True, by_name=False)` and `m.model_dump(by_alias=True)`.

⚠️ Set `populate_by_name=True` (legacy synonym for `validate_by_name=True`) when you want both alias *and* Python field name to validate.

⚠️ Asymmetric defaults: validation uses aliases; serialization does not. May change in V3 — set both `validate_by_alias` and `serialize_by_alias` explicitly.

See [[Pydantic CO 02 - Fields]], [[Pydantic CO 08 - Configuration]], [[Pydantic CO 09 - Serialization]].

💡 **Takeaway:** For mixed naming conventions, set `alias_generator=to_camel` once on the base model, leave Python names snake_case, and dump with `by_alias=True`.
