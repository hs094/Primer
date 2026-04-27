# ER 01 — Error Handling

🔑 **Key insight:** Pydantic raises a single `ValidationError` at the validation boundary that aggregates every problem found, exposing them as structured `ErrorDetails` dicts you can inspect, reshape, or re-render with custom messages.

## ValidationError vs PydanticUserError

`ValidationError` is raised when *data* fails validation against a [[Pydantic CO 01 - Models|model]] / [[Pydantic CO 14 - Type Adapter|TypeAdapter]] — runtime, end-user-data territory. `PydanticUserError` is raised when *the model itself* is misconfigured (decorator misuse, missing annotations, broken discriminators) — see [[Pydantic ER 03 - Usage Errors]].

Inside validators, **raise `ValueError` or `AssertionError`** — Pydantic converts those into entries on the surrounding `ValidationError`. Don't raise `ValidationError` directly from your own validators.

## The shape of an error

`ValidationError` exposes:

- `errors()` → `list[ErrorDetails]`
- `error_count()` → `int`
- `json()` → JSON string of the same payload
- `str(e)` → human-readable summary

Each `ErrorDetails` dict carries:

| key   | meaning                                                          |
|-------|------------------------------------------------------------------|
| `type`| computer-readable identifier (see [[Pydantic ER 02 - Validation Errors]]) |
| `loc` | tuple path to the offending field, e.g. `('items', 1, 'value')`  |
| `msg` | human-readable message                                            |
| `input` | the value that failed                                          |
| `ctx` | optional dict of context values used to render `msg`             |
| `url` | link to the docs page for that error type                        |

## Catching and inspecting

```python
from pydantic import BaseModel, Field, ValidationError, field_validator


class Location(BaseModel):
    lat: float = 0.1
    lng: float = 10.1


class Model(BaseModel):
    is_required: float
    gt_int: int = Field(gt=42)
    list_of_ints: list[int]
    a_float: float
    recursive_model: Location

    @field_validator('a_float', mode='after')
    @classmethod
    def validate_float(cls, value: float) -> float:
        if value > 2.0:
            raise ValueError('Invalid float value')
        return value


data = {
    'list_of_ints': ['1', 2, 'bad'],
    'a_float': 3.0,
    'recursive_model': {'lat': 4.2, 'lng': 'New York'},
    'gt_int': 21,
}

try:
    Model(**data)
except ValidationError as e:
    print(e)
```

⚠️ Validators must raise `ValueError`/`AssertionError` — anything else (including bare `Exception`) bypasses the aggregation machinery and bubbles out as-is.

## Custom messages via `ctx`-aware substitution

You can post-process the structured errors to swap in friendlier copy, formatting with `ctx` values where the original message had placeholders:

```python
from pydantic_core import ErrorDetails

from pydantic import BaseModel, HttpUrl, ValidationError

CUSTOM_MESSAGES = {
    'int_parsing': 'This is not an integer! 🤦',
    'url_scheme': 'Hey, use the right URL scheme! I wanted {expected_schemes}.',
}


def convert_errors(
    e: ValidationError, custom_messages: dict[str, str]
) -> list[ErrorDetails]:
    new_errors: list[ErrorDetails] = []
    for error in e.errors():
        custom_message = custom_messages.get(error['type'])
        if custom_message:
            ctx = error.get('ctx')
            error['msg'] = (
                custom_message.format(**ctx) if ctx else custom_message
            )
        new_errors.append(error)
    return new_errors


class Model(BaseModel):
    a: int
    b: HttpUrl


try:
    Model(a='wrong', b='ftp://example.com')
except ValidationError as e:
    errors = convert_errors(e, CUSTOM_MESSAGES)
    print(errors)
```

The `error['type']` key (`int_parsing`, `url_scheme`, …) is the lookup primitive — every code is catalogued in [[Pydantic ER 02 - Validation Errors]].

## Re-shaping `loc` for API output

`loc` is a tuple by default; many API clients want a dot/bracket path. Walk the tuple yourself:

```python
from typing import Any, Union

from pydantic import BaseModel, ValidationError


def loc_to_dot_sep(loc: tuple[Union[str, int], ...]) -> str:
    path = ''
    for i, x in enumerate(loc):
        if isinstance(x, str):
            if i > 0:
                path += '.'
            path += x
        elif isinstance(x, int):
            path += f'[{x}]'
        else:
            raise TypeError('Unexpected type')
    return path


def convert_errors(e: ValidationError) -> list[dict[str, Any]]:
    new_errors: list[dict[str, Any]] = e.errors()
    for error in new_errors:
        error['loc'] = loc_to_dot_sep(error['loc'])
    return new_errors


class TestNestedModel(BaseModel):
    key: str
    value: str


class TestModel(BaseModel):
    items: list[TestNestedModel]


data = {'items': [{'key': 'foo', 'value': 'bar'}, {'key': 'baz'}]}

try:
    TestModel.model_validate(data)
except ValidationError as e:
    print(e.errors())
    pretty_errors = convert_errors(e)
    print(pretty_errors)
```

This is also the right place to strip `input` from outputs you ship to clients (it can leak the rejected payload).

💡 **Takeaway:** `ValidationError.errors()` is your single, structured source of truth — catch it once at the boundary, transform `type`/`loc`/`msg`/`ctx` to whatever your transport wants.
