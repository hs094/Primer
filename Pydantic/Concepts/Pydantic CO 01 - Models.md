# CO 01 — Models

🔑 **Key insight:** A `BaseModel` subclass is a validating, serializing, schema-generating shell around your annotated attributes — instances are guaranteed to satisfy the schema, not just the type hints.

## Defining a model

```python
from pydantic import BaseModel, ConfigDict

class User(BaseModel):
    id: int
    name: str = 'Jane Doe'
    model_config = ConfigDict(str_max_length=10)

user = User(id='123')
assert user.id == 123
assert user.name == 'Jane Doe'
assert user.model_dump() == {'id': 123, 'name': 'Jane Doe'}
```

`id='123'` is coerced to `int` — Pydantic casts inputs to conform to types unless [[Pydantic CO 13 - Strict Mode]] is on.

## Validation entry points

```python
# Python objects
m = User.model_validate({'id': 123, 'name': 'James'})

# JSON bytes/str — see [[Pydantic CO 04 - JSON]]
m = User.model_validate_json('{"id": 123, "name": 123}')

# All-strings input (e.g. URL params, env)
m = User.model_validate_strings({'id': '123', 'name': 'James'})
```

For non-`BaseModel` types use [[Pydantic CO 14 - Type Adapter]].

## Dumping

```python
m.model_dump()        # dict
m.model_dump_json()   # JSON str
```

`model_dump()` accepts `include=`, `exclude=`, `by_alias=`, `exclude_unset=`, `exclude_defaults=`, `exclude_none=`, `mode='json'|'python'`. See [[Pydantic CO 09 - Serialization]].

## Nested models

```python
class Foo(BaseModel):
    count: int
    size: float | None = None

class Bar(BaseModel):
    apple: str = 'x'

class Spam(BaseModel):
    foo: Foo
    bars: list[Bar]

m = Spam(foo={'count': 4}, bars=[{'apple': 'x1'}])
```

Dicts are recursively coerced into nested model instances.

## Extra data

```python
class Model(BaseModel):
    x: int
    model_config = ConfigDict(extra='allow')

m = Model(x=1, y='a')
assert m.model_dump() == {'x': 1, 'y': 'a'}
```

`extra` is `'ignore'` (default), `'allow'`, or `'forbid'`. See [[Pydantic CO 08 - Configuration]].

## Copy

```python
m = FooBarModel(banana=3.14, foo='hello', bar={'whatever': 123})
print(m.model_copy(update={'banana': 0}))
print(id(m.bar) == id(m.model_copy().bar))        # True (shallow)
print(id(m.bar) == id(m.model_copy(deep=True).bar))  # False (deep)
```

`model_copy()` is shallow by default — `deep=True` for a fresh tree. `update=` does **not** revalidate.

## Construct without validation

```python
m = User.model_construct(id=123, name='John')
```

⚠️ `model_construct()` skips validation entirely. Only use with already-trusted data (e.g. rows just read from your own DB). Garbage in, garbage out.

## Generic models

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

DataT = TypeVar('DataT')

class Response(BaseModel, Generic[DataT]):
    data: DataT

print(Response[int](data=1))
print(Response[str](data='value'))
```

⚠️ `isinstance(x, Response[int])` is not reliable — parametrized generics aren't real classes for `isinstance` purposes.

## RootModel — non-dict roots

```python
from pydantic import RootModel

Pets = RootModel[list[str]]
print(Pets(['dog', 'cat']))
#> root=['dog', 'cat']
```

Use when the top-level JSON is a list, scalar, or union — not a mapping.

## Dynamic model creation

```python
from typing import Annotated
from pydantic import Field, create_model

DynamicModel = create_model(
    'DynamicModel',
    foo=(str, Field(alias='FOO')),
    bar=Annotated[str, Field(description='Bar field')]
)
```

Each field is `(type, default)` or an `Annotated[...]` form. Useful for codegen and runtime-driven schemas.

## Faux immutability

```python
class FooBarModel(BaseModel):
    model_config = ConfigDict(frozen=True)
    a: str

foobar = FooBarModel(a='hello')
# foobar.a = 'different'  -> ValidationError
```

⚠️ Immutability is enforced at the `__setattr__` level only — mutable fields (lists, dicts) can still be mutated in place.

## Private attributes

```python
from datetime import datetime
from pydantic import BaseModel, PrivateAttr

class TimeAwareModel(BaseModel):
    _processed_at: datetime = PrivateAttr(default_factory=datetime.now)
    _secret_value: str
```

Names starting with `_` are private — excluded from validation, serialization, and the model signature.

## Abstract models

`BaseModel` plays nicely with `abc.ABC`. Mark a parent abstract; concrete subclasses inherit fields and must implement abstract methods like any ABC.

## Helpful methods reference

| Method | Purpose |
|---|---|
| `model_validate(obj)` | validate Python object |
| `model_validate_json(b)` | validate JSON bytes/str |
| `model_validate_strings(d)` | all-strings input mode |
| `model_dump()` | to dict |
| `model_dump_json()` | to JSON str |
| `model_copy()` | shallow/deep copy with update |
| `model_construct()` | unvalidated construction |
| `model_json_schema()` | JSON Schema — see [[Pydantic CO 03 - JSON Schema]] |
| `model_fields` | `dict[str, FieldInfo]` — see [[Pydantic CO 02 - Fields]] |
| `model_rebuild()` | resolve forward refs — see [[Pydantic CO 12 - Forward Annotations]] |

💡 **Takeaway:** `BaseModel` is a contract — define types, then trust validated instances; reach for `model_construct` only when you've already paid the validation tax elsewhere.
