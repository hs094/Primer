# CO 19 — Experimental

🔑 **Key insight:** Anything under `pydantic.experimental` is a preview — usable today, but the API can shift between minor releases. Pin pydantic if you depend on these.

⚠️ Experimental features are explicitly **not covered by semver** — surface area can change or vanish entirely. Don't bake them deep into stable interfaces without an abstraction layer.

## Pipeline API

`pydantic.experimental.pipeline` lets you compose validation, transformation, and predicate steps in a type-safe chain. It maps onto the existing `BeforeValidator` / `AfterValidator` / `WrapValidator` machinery — see [[Pydantic CO 10 - Validators]].

```python
from __future__ import annotations

from datetime import datetime
from typing import Annotated

from pydantic import BaseModel
from pydantic.experimental.pipeline import validate_as


class User(BaseModel):
  name: Annotated[str, validate_as(str).str_lower()]
  age: Annotated[int, validate_as(int).gt(0)]
  username: Annotated[str, validate_as(str).str_pattern(r'[a-z]+')]
  password: Annotated[
      str,
      validate_as(str)
      .transform(str.lower)
      .predicate(lambda x: x != 'password'),
  ]
  favorite_number: Annotated[
      int,
      (validate_as(int) | validate_as(str).str_strip().validate_as(int)).gt(
          0
      ),
  ]
  friends: Annotated[list[User], validate_as(...).len(0, 100)]
  bio: Annotated[
      datetime,
      validate_as(int)
      .transform(lambda x: x / 1_000_000)
      .validate_as(...),
  ]
```

### Mapping to standard validators

```python
from typing import Annotated

from pydantic.experimental.pipeline import transform, validate_as

# BeforeValidator
Annotated[int, validate_as(str).str_strip().validate_as(...)]

# AfterValidator
Annotated[int, transform(lambda x: x * 2)]

# WrapValidator
Annotated[
  int,
  validate_as(str)
  .str_strip()
  .validate_as(...)
  .transform(lambda x: x * 2),
]
```

### When *not* to use it

The docs themselves note that plain Python is often clearer:

```python
from __future__ import annotations

from pydantic import BaseModel


class UserIn(BaseModel):
    favorite_number: int | str


class UserOut(BaseModel):
    favorite_number: int


def my_api(user: UserIn) -> UserOut:
    favorite_number = user.favorite_number
    if isinstance(favorite_number, str):
        favorite_number = int(user.favorite_number.strip())

    return UserOut(favorite_number=favorite_number)


assert my_api(UserIn(favorite_number=' 1 ')).favorite_number == 1
```

## Partial validation

Introduced in v2.10. Validate **incomplete** JSON — useful for streaming LLM output. Pass `experimental_allow_partial=True` to `TypeAdapter` validation methods.

```python
from typing import Annotated

from annotated_types import MinLen
from typing_extensions import NotRequired, TypedDict

from pydantic import TypeAdapter


class Foobar(TypedDict):
  a: int
  b: NotRequired[float]
  c: NotRequired[Annotated[str, MinLen(5)]]


ta = TypeAdapter(list[Foobar])

v = ta.validate_json('[{"a": 1, "b"', experimental_allow_partial=True)
print(v)
#> [{'a': 1}]

v = ta.validate_json(
  '[{"a": 1, "b": 1.0, "c": "abcd', experimental_allow_partial=True
)
print(v)
#> [{'a': 1, 'b': 1.0}]

v = ta.validate_json(
  '[{"b": 1.0, "c": "abcde"', experimental_allow_partial=True
)
print(v)
#> []

v = ta.validate_python([{'a': 1}], experimental_allow_partial=True)
print(v)
#> [{'a': 1}]
```

### Trailing strings mode

```python
v = ta.validate_json(
  '[{"a": 1, "b": 1.0, "c": "abcdefg',
  experimental_allow_partial='trailing-strings',
)
print(v)
#> [{'a': 1, 'b': 1.0, 'c': 'abcdefg'}]
```

Modes: `False`/`'off'` (default), `True`/`'on'`, `'trailing-strings'`.

### Caveats

⚠️ **TypeAdapter only** — not available via `BaseModel`. Supported types: `list`, `set`, `frozenset`, `dict`, `TypedDict`.

⚠️ **Errors in the last element are silently dropped** — a footgun if you assume "validated" means "complete":

```python
from typing import Annotated

from annotated_types import Ge

from pydantic import TypeAdapter

ta = TypeAdapter(list[Annotated[int, Ge(10)]])
v = ta.validate_python([20, 30, 4], experimental_allow_partial=True)
print(v)
#> [20, 30]

ta = TypeAdapter(list[int])
v = ta.validate_python([1, 2, 'wrong'], experimental_allow_partial=True)
print(v)
#> [1, 2]
```

⚠️ **Some invalid-but-complete JSON is accepted** as if it were partial:

```python
from typing import Annotated
from annotated_types import MinLen
from typing_extensions import TypedDict
from pydantic import TypeAdapter

class Foobar(TypedDict, total=False):
  a: int
  b: Annotated[str, MinLen(5)]

ta = TypeAdapter(Foobar)
v = ta.validate_json('{"a": 1, "b": "12"}', experimental_allow_partial=True)
print(v)
#> {'a': 1}
```

## Validating a callable's arguments

`pydantic.experimental.arguments_schema.generate_arguments_schema` builds a schema for a callable's signature **without invoking it** — handy for loading args from JSON/RPC payloads.

```python
from pydantic_core import SchemaValidator

from pydantic.experimental.arguments_schema import generate_arguments_schema


def func(p: bool, *args: str, **kwargs: int) -> None: ...


arguments_schema = generate_arguments_schema(func=func)

val = SchemaValidator(arguments_schema, config={'coerce_numbers_to_str': True})

args, kwargs = val.validate_json(
  '{"p": true, "args": ["arg1", 1], "kwargs": {"extra": 1}}'
)
print(args, kwargs)
#> (True, 'arg1', '1') {'extra': 1}
```

Pair with `inspect.signature(...).bind(...)` if you want the bound mapping:

```python
from inspect import signature

signature(func).bind(*args, **kwargs).arguments
#> {'p': True, 'args': ('arg1', '1'), 'kwargs': {'extra': 1}}
```

### Skipping parameters

```python
from typing import Any

from pydantic_core import SchemaValidator

from pydantic.experimental.arguments_schema import generate_arguments_schema


def func(p: bool, *args: str, **kwargs: int) -> None: ...


def skip_first_parameter(index: int, name: str, annotation: Any) -> Any:
    if index == 0:
        return 'skip'


arguments_schema = generate_arguments_schema(
    func=func,
    parameters_callback=skip_first_parameter,
)

val = SchemaValidator(arguments_schema)

args, kwargs = val.validate_json('{"args": ["arg1"], "kwargs": {"extra": 1}}')
print(args, kwargs)
#> ('arg1',) {'extra': 1}
```

## `MISSING` sentinel

A first-class "this field was never set" marker — distinct from `None`. Fields holding `MISSING` are excluded from serialization output and from JSON Schema.

```python
from typing import Union

from pydantic import BaseModel
from pydantic.experimental.missing_sentinel import MISSING


class Configuration(BaseModel):
    timeout: Union[int, None, MISSING] = MISSING


# configuration defaults, stored somewhere else:
defaults = {'timeout': 200}

conf = Configuration()

# `timeout` is excluded from the serialization output:
conf.model_dump()
# {}

# The `MISSING` value doesn't appear in the JSON Schema:
Configuration.model_json_schema()['properties']['timeout']
#> {'anyOf': [{'type': 'integer'}, {'type': 'null'}], 'title': 'Timeout'}}


# `is` can be used to discriminate between the sentinel and other values:
timeout = conf.timeout if conf.timeout is not MISSING else defaults['timeout']
```

⚠️ Limitations: relies on draft **PEP 661**; static-type checking needs **Pyright 1.1.402+**; **pickling unsupported**.

## Stability summary

| Feature | Availability | Note |
|---|---|---|
| Pipeline API | preview | maps to standard validators |
| Partial validation | v2.10+ | `TypeAdapter` only; trims last-element errors |
| `generate_arguments_schema` | preview | validates callable args without calling |
| `MISSING` sentinel | preview | depends on PEP 661 |

## Cross-references

- [[Pydantic CO 10 - Validators]] — what pipeline desugars into.
- [[Pydantic CO 14 - Type Adapter]] — the host for partial validation.
- [[Pydantic CO 09 - Serialization]] — `MISSING` and dump behaviour.
- [[Pydantic IT 02 - Resolving Annotations]] — how schemas are derived.
- [[Pydantic IN 01 - Logfire]] — observability for partial / streaming validation.

💡 **Takeaway:** Use experimental features behind a thin adapter — they unblock real problems (LLM streams, signature validation), but expect to revisit them on every minor bump.
