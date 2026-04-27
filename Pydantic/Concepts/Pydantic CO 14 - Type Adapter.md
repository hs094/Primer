# CO 14 — Type Adapter

🔑 **Key insight:** `TypeAdapter` gives you Pydantic's full validate/dump/schema pipeline for any type — `list[int]`, `dict[str, User]`, `TypedDict`, dataclasses, generics — without the ceremony of wrapping it in a `BaseModel`.

## Core idea

You may have types that aren't `BaseModel` subclasses but you still want validation against. `TypeAdapter` builds a schema from any type expression and exposes the same methods you know from [[Pydantic CO 01 - Models]]: `validate_python`, `validate_json`, `dump_python`, `dump_json`, `json_schema`.

## Validating a list of `TypedDict`s

```python
from typing_extensions import TypedDict

from pydantic import TypeAdapter, ValidationError


class User(TypedDict):
    name: str
    id: int


user_list_adapter = TypeAdapter(list[User])
user_list = user_list_adapter.validate_python([{'name': 'Fred', 'id': '3'}])
print(repr(user_list))
#> [{'name': 'Fred', 'id': 3}]

try:
    user_list_adapter.validate_python(
        [{'name': 'Fred', 'id': 'wrong', 'other': 'no'}]
    )
except ValidationError as e:
    print(e)
    """
    1 validation error for list[User]
    0.id
      Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='wrong', input_type=str]
    """

print(repr(user_list_adapter.dump_json(user_list)))
#> b'[{"name":"Fred","id":3}]'
```

⚠️ `dump_json` returns **bytes**, not a string — different from `BaseModel.model_dump_json` which returns `str`. The docs preserve this for back-compat.

## Validating a list of `BaseModel`s

```python
from pydantic import BaseModel, TypeAdapter


class Item(BaseModel):
    id: int
    name: str


# `item_data` could come from an API call, eg., via something like:
# item_data = requests.get('https://my-api.com/items').json()
item_data = [{'id': 1, 'name': 'My Item'}]

items = TypeAdapter(list[Item]).validate_python(item_data)
print(items)
#> [Item(id=1, name='My Item')]
```

This is the pattern for "JSON array of models" — common at API boundaries. No need for a `RootModel` wrapper.

## When *not* to use `TypeAdapter`

- **Don't use it as a field annotation** in a `BaseModel` — use [[Pydantic CO 01 - Models]] root models / nested models. From the docs: it "should not be used as a type annotation for specifying fields of a `BaseModel`".
- **Don't use it for shapes you'll attach methods/validators to** — that's what `BaseModel` is for.

Reach for it when the data shape is already expressible in a single type expression and you just need validate/dump/schema.

## Methods at a glance

```python
ta = TypeAdapter(list[User])

ta.validate_python(obj)     # parse Python data
ta.validate_json(raw)       # parse JSON bytes/str
ta.dump_python(value)       # → Python primitives
ta.dump_json(value)         # → bytes
ta.json_schema()            # JSON Schema dict
```

See [[Pydantic CO 09 - Serialization]] for dump options (`mode`, `exclude`, `by_alias`).

## Performance: instantiate once, reuse

Building a `TypeAdapter` runs schema analysis with **non-trivial overhead**. Best practice: create at module scope, reuse across requests.

```python
# module-level
_USER_LIST = TypeAdapter(list[User])

def parse_users(raw: bytes) -> list[User]:
    return _USER_LIST.validate_json(raw)
```

See [[Pydantic CO 18 - Performance]].

## Deferred schema building (v2.10+)

For forward references or expensive schemas, defer construction:

```python
from pydantic import ConfigDict, TypeAdapter

ta = TypeAdapter('MyInt', config=ConfigDict(defer_build=True))

# some time later, the forward reference is defined
MyInt = int

ta.rebuild()
assert ta.validate_python(1) == 1
```

Same idea as `model_rebuild()` — see [[Pydantic CO 12 - Forward Annotations]].

## With dataclasses

Since [[Pydantic CO 11 - Dataclasses]] don't expose `model_dump`/`model_validate`, `TypeAdapter` is how you dump/validate them:

```python
from pydantic import TypeAdapter
from pydantic.dataclasses import dataclass

@dataclass
class Foo:
    f: int

TypeAdapter(Foo).dump_python(Foo(f=1))      # {'f': 1}
TypeAdapter(Foo).validate_python({'f': 1})  # Foo(f=1)
```

## With strict mode

`TypeAdapter` honors per-call strict — see [[Pydantic CO 13 - Strict Mode]]:

```python
TypeAdapter(date).validate_python('2000-01-01', strict=True)
```

## Common shapes

```python
TypeAdapter(int)
TypeAdapter(list[int])
TypeAdapter(dict[str, User])
TypeAdapter(tuple[int, str, bool])
TypeAdapter(User | Admin)
TypeAdapter(Literal['a', 'b'])
```

Anything you can write as a type expression, `TypeAdapter` can validate against.

💡 **Takeaway:** When the data shape doesn't deserve a class, give it a `TypeAdapter` — same validation power, zero `BaseModel` boilerplate.
