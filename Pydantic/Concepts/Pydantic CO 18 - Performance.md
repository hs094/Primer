# CO 18 — Performance

🔑 **Key insight:** Pydantic is fast by default; the wins come from **avoiding redundant work** — validate JSON directly, reuse `TypeAdapter`, use discriminated unions, and don't hand the validator more freedom than it needs.

## Validate JSON directly, not via `json.loads`

Skip the intermediate Python `dict`:

```python
# slower — two passes
Model.model_validate(json.loads(payload))

# faster — single pass in Rust
Model.model_validate_json(payload)
```

The two-step path is only competitive when you have `'before'` or `'wrap'` validators on the model.

## Reuse `TypeAdapter`

Each `TypeAdapter(...)` call builds a fresh validator + serializer. Build it **once**.

```python
# Bad — rebuilds on every call
from pydantic import TypeAdapter

def my_func():
    adapter = TypeAdapter(list[int])
    # do something with adapter
```

```python
# Good — built once, reused
from pydantic import TypeAdapter

adapter = TypeAdapter(list[int])

def my_func():
    ...
    # do something with adapter
```

See [[Pydantic CO 14 - Type Adapter]].

## Prefer concrete types

`Sequence`, `Mapping`, `Iterable` force the validator to try multiple shapes via `isinstance` checks. Use `list`, `tuple`, or `dict` whenever the abstract isn't load-bearing.

```python
# Slower
items: Sequence[int]
config: Mapping[str, str]

# Faster
items: list[int]
config: dict[str, str]
```

## `Any` skips validation

When the value is already trusted (e.g. internal handoff), `Any` short-circuits the whole machinery.

```python
from typing import Any
from pydantic import BaseModel

class Model(BaseModel):
    a: Any

model = Model(a=1)
```

## Don't subclass primitives — compose

Carrying extra state on a `str`/`int` subclass forces dynamic checks Pydantic can't optimise.

```python
# Don't
class CompletedStr(str):
    def __init__(self, s: str):
        self.s = s
        self.done = False
```

```python
# Do
from pydantic import BaseModel

class CompletedModel(BaseModel):
    s: str
    done: bool = False
```

## Tagged unions over plain unions

A discriminator makes the union an O(1) dispatch instead of a linear "try each variant" walk. See [[Pydantic CO 06 - Unions]].

```python
from typing import Any, Literal
from pydantic import BaseModel, Field

class DivModel(BaseModel):
    el_type: Literal['div'] = 'div'
    class_name: str | None = None
    children: list[Any] | None = None

class SpanModel(BaseModel):
    el_type: Literal['span'] = 'span'
    class_name: str | None = None
    contents: str | None = None

class ButtonModel(BaseModel):
    el_type: Literal['button'] = 'button'
    class_name: str | None = None
    contents: str | None = None

class InputModel(BaseModel):
    el_type: Literal['input'] = 'input'
    class_name: str | None = None
    value: str | None = None

class Html(BaseModel):
    contents: DivModel | SpanModel | ButtonModel | InputModel = Field(
        discriminator='el_type'
    )
```

## `TypedDict` over nested `BaseModel`

For pure data, `TypedDict` skips per-instance object construction. Roughly **~2.5× faster** in the docs' benchmark.

```python
from timeit import timeit
from typing_extensions import TypedDict
from pydantic import BaseModel, TypeAdapter

class A(TypedDict):
    a: str
    b: int

class TypedModel(TypedDict):
    a: A

class B(BaseModel):
    a: str
    b: int

class Model(BaseModel):
    b: B

ta = TypeAdapter(TypedModel)
result1 = timeit(
    lambda: ta.validate_python({'a': {'a': 'a', 'b': 2}}), number=10000
)
result2 = timeit(
    lambda: Model.model_validate({'b': {'a': 'a', 'b': 2}}), number=10000
)
print(result2 / result1)
```

## Avoid wrap validators on hot paths

`'wrap'` validators materialise data in Python, breaking the all-Rust path. Reserve for genuinely complex logic; prefer `'before'` / `'after'` / `'plain'` otherwise. See [[Pydantic CO 10 - Validators]].

## `FailFast` for sequences

Stops on the first item failure — trade error completeness for speed.

```python
from typing import Annotated
from pydantic import FailFast, TypeAdapter, ValidationError

ta = TypeAdapter(Annotated[list[bool], FailFast()])
try:
    ta.validate_python([True, 'invalid', False, 'also invalid'])
except ValidationError as exc:
    print(exc)
    """
    1 validation error for list[bool]
    1
      Input should be a valid boolean, unable to interpret input [type=bool_parsing, input_value='invalid', input_type=str]
    """
```

⚠️ With `FailFast`, you lose visibility into all subsequent errors — only the first item failure surfaces.

## Quick checklist

| Tip | Win |
|---|---|
| `model_validate_json` over `model_validate(json.loads(...))` | Single Rust pass |
| Reuse `TypeAdapter` | Avoid re-building schemas |
| `list`/`dict` over `Sequence`/`Mapping` | Skip abstract isinstance trees |
| `Any` for trusted values | Zero validation |
| Discriminated unions | O(1) variant dispatch |
| `TypedDict` for plain data | ~2.5× over nested `BaseModel` |
| Avoid wrap validators | Stay in Rust |
| `FailFast` on big sequences | Short-circuit |

## Further reading

- [[Pydantic CO 14 - Type Adapter]] — caching adapter instances.
- [[Pydantic CO 06 - Unions]] — discriminator setup.
- [[Pydantic CO 10 - Validators]] — picking the right validator mode.
- [[Pydantic CO 19 - Experimental]] — partial validation, pipeline.
- [[Pydantic IT 01 - Architecture]] — why `pydantic-core` (Rust) is the engine.

💡 **Takeaway:** Stay in Rust — narrow types, build adapters once, validate JSON directly, and use discriminators wherever a union is on the hot path.
