# CO 04 — JSON

🔑 **Key insight:** `model_validate_json()` parses JSON directly through `jiter` (Rust) into validated Pydantic objects in one pass — never deserialize-then-validate.

## Parsing JSON straight to a model

```python
from datetime import date
from pydantic import BaseModel, ConfigDict, ValidationError

class Event(BaseModel):
    model_config = ConfigDict(strict=True)

    when: date
    where: tuple[int, int]

json_data = '{"when": "1987-01-28", "where": [51, -1]}'
print(Event.model_validate_json(json_data))
#> when=datetime.date(1987, 1, 28) where=(51, -1)

try:
    Event.model_validate({'when': '1987-01-28', 'where': [51, -1]})
except ValidationError as e:
    print(e)
```

Note: even with `strict=True`, JSON parsing still maps `"1987-01-28"` to `date` because that string-to-date is a JSON-mode coercion, not a Python-mode coercion. See [[Pydantic CO 16 - Conversion Table]] and [[Pydantic CO 13 - Strict Mode]].

## Three entry points

| Function | When |
|---|---|
| `BaseModel.model_validate_json(s)` | a model |
| `TypeAdapter(T).validate_json(s)` | any annotated type — see [[Pydantic CO 14 - Type Adapter]] |
| `pydantic_core.from_json(s)` | raw JSON → Python (no validation) |

## Partial JSON parsing (v2.7+)

```python
from pydantic_core import from_json

partial_json_data = '["aa", "bb", "c'
result = from_json(partial_json_data, allow_partial=True)
print(result)
#> ['aa', 'bb']
```

Trailing incomplete tokens are dropped:

```python
from pydantic_core import from_json

partial_dog_json = '{"breed": "lab", "name": "fluffy", "friends": ["buddy", "spot", "rufus"], "age'
dog_dict = from_json(partial_dog_json, allow_partial=True)
print(dog_dict)
#> {'breed': 'lab', 'name': 'fluffy', 'friends': ['buddy', 'spot', 'rufus']}
```

Plug into a model:

```python
from pydantic_core import from_json
from pydantic import BaseModel

class Dog(BaseModel):
    breed: str
    name: str
    friends: list

partial_dog_json = '{"breed": "lab", "name": "fluffy", "friends": ["buddy", "spot", "rufus"], "age'
dog = Dog.model_validate(from_json(partial_dog_json, allow_partial=True))
print(repr(dog))
#> Dog(breed='lab', name='fluffy', friends=['buddy', 'spot', 'rufus'])
```

⚠️ Partial parsing is intended for streaming LLM output and similar. For a partial dict to validate, missing fields must have defaults — otherwise you'll get `missing` errors.

### Defaulting missing fields with a wrap validator

```python
from typing import Annotated, Any, Optional
import pydantic_core
from pydantic import BaseModel, ValidationError, WrapValidator

def default_on_error(v, handler) -> Any:
    """Raise PydanticUseDefault if the value is missing."""
    try:
        return handler(v)
    except ValidationError as exc:
        if all(e['type'] == 'missing' for e in exc.errors()):
            raise pydantic_core.PydanticUseDefault()
        else:
            raise

class NestedModel(BaseModel):
    x: int
    y: str

class MyModel(BaseModel):
    foo: Optional[str] = None
    bar: Annotated[Optional[tuple[str, int]], WrapValidator(default_on_error)] = None
    nested: Annotated[Optional[NestedModel], WrapValidator(default_on_error)] = None

m = MyModel.model_validate(
    pydantic_core.from_json('{"foo": "x", "bar": ["world",', allow_partial=True)
)
print(repr(m))
#> MyModel(foo='x', bar=None, nested=None)
```

See [[Pydantic CO 10 - Validators]] for `WrapValidator`.

## String caching (v2.7+)

`from_json(..., cache_strings=...)` controls string interning during parse:

| Value | Behavior |
|---|---|
| `True` / `'all'` (default) | cache all strings |
| `'keys'` | cache only mapping keys |
| `False` / `'none'` | disabled |

Only strings under 64 chars are cached, in a 16,384-entry cache. Big wins on payloads with repeated keys; tiny cost when keys are mostly unique.

## Serialization

```python
m.model_dump_json()
TypeAdapter(T).dump_json(value)
pydantic_core.to_json(value)
```

`model_dump_json()` accepts the same `include`/`exclude`/`by_alias`/`exclude_unset`/`exclude_none` as `model_dump()`, plus `indent=`. Output is `bytes` for `to_json` / `dump_json` and `str` for `model_dump_json`. Full surface in [[Pydantic CO 09 - Serialization]].

## `Json[T]` — embedded JSON strings as fields

```python
from pydantic import BaseModel, Json

class Wrapper(BaseModel):
    payload: Json[list[int]]

w = Wrapper(payload='[1, 2, 3]')
assert w.payload == [1, 2, 3]
```

`Json[T]` says "this field arrives as a JSON-encoded string; parse and then validate as `T`." Schema renders as `{"format": "json-string"}` — see [[Pydantic CO 03 - JSON Schema]].

## Why prefer `model_validate_json` over `json.loads` + `model_validate`

- One pass through the bytes (jiter parses straight into core schema slots).
- JSON-mode coercions kick in (e.g. `"1987-01-28"` → `date`) which Python mode rejects under `strict=True`.
- Better error locations — column/line context from the parser.

💡 **Takeaway:** Treat JSON as a first-class input — use `model_validate_json`, lean on `from_json(allow_partial=True)` for streaming, and reach for `Json[T]` when JSON is nested inside JSON.
