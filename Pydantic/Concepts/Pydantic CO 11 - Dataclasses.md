# CO 11 — Dataclasses

🔑 **Key insight:** `pydantic.dataclasses.dataclass` bolts validation onto stdlib-shaped classes, but it is **not** a `BaseModel` replacement — it has no `model_dump`, no `model_validate`; you reach for `TypeAdapter` to do anything beyond construction.

## Decorator

```python
from datetime import datetime
from typing import Optional

from pydantic.dataclasses import dataclass


@dataclass
class User:
    id: int
    name: str = 'John Doe'
    signup_ts: Optional[datetime] = None


user = User(id='42', signup_ts='2032-06-21T12:00')
# Returns: User(id=42, name='John Doe', signup_ts=datetime.datetime(2032, 6, 21, 12, 0))
```

Coercion fires at `__init__` — `'42'` becomes `42` exactly like in [[Pydantic CO 01 - Models]].

## When to use vs `BaseModel`

Pydantic dataclasses are **not** a replacement for [[Pydantic CO 01 - Models]]. Differences:

- **No built-in methods** — validation, dumping, and JSON Schema generation require wrapping with `TypeAdapter`.
- **Limited validator support** — different approach to field and model validators.
- **Extra data handling** — extras aren't included in serialization, no `__pydantic_extra__` customization.
- **Generic dataclasses** — `Foo[int]` is a generic alias, not a proper type object; treated the same as `Foo[Any]`.

Use them when you must integrate with code that expects stdlib `dataclasses` semantics (`dataclasses.asdict`, `dataclasses.fields`, structural-typing libs). Otherwise: `BaseModel`.

## Configuration

Two equivalent forms — see [[Pydantic CO 08 - Configuration]] for keys.

```python
from pydantic import ConfigDict
from pydantic.dataclasses import dataclass

@dataclass(config=ConfigDict(validate_assignment=True))
class MyDataclass1:
    a: int
```

```python
@dataclass
class MyDataclass2:
    a: int
    __pydantic_config__ = ConfigDict(validate_assignment=True)
```

## Fields: `Field()` and `dataclasses.field()` both work

```python
import dataclasses
from typing import Optional
from pydantic import Field
from pydantic.dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str = 'John Doe'
    friends: list[int] = dataclasses.field(default_factory=lambda: [0])
    age: Optional[int] = dataclasses.field(
        default=None,
        metadata={'title': 'The age of the user', 'description': 'do not lie!'},
    )
    height: Optional[int] = Field(
        default=None, title='The height in cm', ge=50, le=300
    )
```

See [[Pydantic CO 02 - Fields]] for the full constraint set.

## Validators

```python
from pydantic import field_validator
from pydantic.dataclasses import dataclass

@dataclass
class DemoDataclass:
    product_id: str

    @field_validator('product_id', mode='before')
    @classmethod
    def convert_int_serial(cls, v):
        if isinstance(v, int):
            v = str(v).zfill(5)
        return v
```

⚠️ With dataclasses, the `values` parameter in `before` model validators uses `ArgsKwargs` — **not** a dict like with `BaseModel`. See [[Pydantic CO 10 - Validators]].

## TypeAdapter is mandatory for dump / validate

```python
from pydantic import TypeAdapter
from pydantic.dataclasses import dataclass

@dataclass
class Foo:
    f: int

TypeAdapter(Foo).dump_python(Foo(f=1))  # {'f': 1}
TypeAdapter(Foo).validate_python({'f': 1})  # Foo(f=1)
```

See [[Pydantic CO 14 - Type Adapter]].

## Stdlib dataclass interop

Pydantic validates inherited fields from stdlib dataclasses automatically:

```python
import dataclasses
import pydantic

@dataclasses.dataclass
class Z:
    z: int

@pydantic.dataclasses.dataclass
class X(Z):
    x: int = 0

foo = X(x=b'1', z='3')  # Validation applied to inherited field
```

The decorator can also retrofit an existing stdlib dataclass:

```python
@dataclasses.dataclass
class A:
    a: int

PydanticA = pydantic.dataclasses.dataclass(A)
print(PydanticA(a='1'))  # A(a=1)
```

When a stdlib dataclass is used **inside** a `BaseModel`, validation is applied with the model's configuration. For arbitrary types as fields:

```python
class Model(BaseModel):
    model_config = ConfigDict(arbitrary_types_allowed=True)
    dc: DC
    other: str
```

## Distinguishing the two flavors

```python
import pydantic

print(pydantic.dataclasses.is_pydantic_dataclass(PydanticDataclass))  # True
print(pydantic.dataclasses.is_pydantic_dataclass(StdLibDataclass))    # False
```

## `__post_init__` ordering

`__post_init__` runs **between** the before- and after-model validators:

```python
@dataclass
class User:
    birth: Birth

    @model_validator(mode='before')
    @classmethod
    def before(cls, values):
        # Executes first
        return values

    def __post_init__(self):
        # Executes second
        ...

    @model_validator(mode='after')
    def after(self):
        # Executes third
        return self
```

## Schema rebuilding

`rebuild_dataclass()` reconstructs the core schema for forward refs — the dataclass analog of `model_rebuild()`. See [[Pydantic CO 12 - Forward Annotations]] and [[Pydantic IT 02 - Resolving Annotations]].

💡 **Takeaway:** Reach for `pydantic.dataclasses.dataclass` only when stdlib-shape compatibility matters; everywhere else `BaseModel` gives you more for less ceremony.
