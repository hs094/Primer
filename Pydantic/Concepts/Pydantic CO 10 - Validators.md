# CO 10 — Validators

🔑 **Key insight:** Mode picks *when* you run (before / after / wrap / plain Pydantic's own validation), and the annotated form picks *where* the rule lives — type-bound and reusable, or class-bound and multi-field.

## Field validator modes

| Mode | Sees | Returns | Pydantic's own validation |
| --- | --- | --- | --- |
| `before` | raw input (`Any`) | value passed onward | runs *after* you |
| `after` | already-parsed, type-correct value | value passed onward | runs *before* you |
| `plain` | raw input | final value | **skipped** |
| `wrap` | raw input + `handler` | final value | runs only if you call `handler` |

## After — type-safe, runs last

Value is already parsed by Pydantic; returning a different value mutates the field. Most common mode.

```python
from typing import Annotated

from pydantic import AfterValidator, BaseModel, field_validator


def is_even(value: int) -> int:
    if value % 2 == 1:
        raise ValueError(f'{value} is not an even number')
    return value


class Model(BaseModel):
    number: Annotated[int, AfterValidator(is_even)]


# decorator equivalent:
class Model2(BaseModel):
    number: int

    @field_validator('number', mode='after')
    @classmethod
    def is_even(cls, value: int) -> int:
        if value % 2 == 1:
            raise ValueError(f'{value} is not an even number')
        return value
```

## Before — coerce raw input

```python
from typing import Annotated, Any

from pydantic import BaseModel, BeforeValidator


def ensure_list(value: Any) -> Any:
    if not isinstance(value, list):
        return [value]
    return value


class Model(BaseModel):
    numbers: Annotated[list[int], BeforeValidator(ensure_list)]


print(Model(numbers=2))
#> numbers=[2]
```

⚠️ Avoid mutating the value if you might raise a validation error later.

## Plain — terminate validation

Pydantic does *no* further parsing against the field type:

```python
from typing import Annotated, Any

from pydantic import BaseModel, PlainValidator


def val_number(value: Any) -> Any:
    if isinstance(value, int):
        return value * 2
    else:
        return value


class Model(BaseModel):
    number: Annotated[int, PlainValidator(val_number)]


print(Model(number='invalid'))
#> number='invalid'
```

⚠️ Plain validators bypass type checking — that `'invalid'` string lives inside an `int` field.

## Wrap — recover from errors

```python
from typing import Annotated, Any

from pydantic import (
    BaseModel,
    Field,
    ValidationError,
    ValidatorFunctionWrapHandler,
    WrapValidator,
)


def truncate(value: Any, handler: ValidatorFunctionWrapHandler) -> str:
    try:
        return handler(value)
    except ValidationError as err:
        if err.errors()[0]['type'] == 'string_too_long':
            return handler(value[:5])
        else:
            raise


class Model(BaseModel):
    my_string: Annotated[str, Field(max_length=5), WrapValidator(truncate)]


print(Model(my_string='abcdef'))
#> my_string='abcde'
```

## Annotated vs decorator

Annotated form makes validators reusable across models:

```python
EvenNumber = Annotated[int, AfterValidator(is_even)]


class Model1(BaseModel):
    my_number: EvenNumber


class Model3(BaseModel):
    list_of_even_numbers: list[EvenNumber]
```

Decorator form binds one rule to many fields on one model:

```python
class Model(BaseModel):
    f1: str
    f2: str

    @field_validator('f1', 'f2', mode='before')
    @classmethod
    def capitalize(cls, value: str) -> str:
        return value.capitalize()
```

`'*'` applies to every field. `check_fields=False` skips the "is this field declared" check (useful on mixins).

## Model validators

`@model_validator(mode='after')` is an instance method that **must return `self`** — forgetting silently yields `None`. `mode='before'` is a classmethod on raw `data`. `mode='wrap'` takes `(data, handler)` for full control.

```python
from typing import Any
from typing_extensions import Self

from pydantic import BaseModel, model_validator


class UserModel(BaseModel):
    username: str
    password: str
    password_repeat: str

    @model_validator(mode='before')
    @classmethod
    def reject_card_number(cls, data: Any) -> Any:
        if isinstance(data, dict) and 'card_number' in data:
            raise ValueError("'card_number' should not be included")
        return data

    @model_validator(mode='after')
    def check_passwords_match(self) -> Self:
        if self.password != self.password_repeat:
            raise ValueError('Passwords do not match')
        return self
```

Wrap form:

```python
from pydantic import ModelWrapValidatorHandler, ValidationError


class UserModel(BaseModel):
    username: str

    @model_validator(mode='wrap')
    @classmethod
    def log_failed(
        cls, data: Any, handler: ModelWrapValidatorHandler[Self]
    ) -> Self:
        try:
            return handler(data)
        except ValidationError:
            logging.error('Model %s failed to validate with data %s', cls, data)
            raise
```

A model validator on a base class runs for subclass validation; same-name override in a subclass replaces it.

## Raising errors

`raise ValueError(...)` — most common. `assert` works but is **skipped under `python -O`**.

`PydanticCustomError` for structured errors with type/template/context (see [[Pydantic ER 02 - Validation Errors]], [[Pydantic ER 01 - Error Handling]]):

```python
from pydantic_core import PydanticCustomError

from pydantic import BaseModel, field_validator


class Model(BaseModel):
    x: int

    @field_validator('x', mode='after')
    @classmethod
    def validate_x(cls, v: int) -> int:
        if v % 42 == 0:
            raise PydanticCustomError(
                'the_answer_error',
                '{number} is the answer!',
                {'number': v},
            )
        return v
```

## ValidationInfo

Optional last argument on any validator. `info.data` holds already-validated peers (only those declared *before* this field — order matters). `info.context` holds the dict you passed to `model_validate(..., context=...)`.

```python
from pydantic import BaseModel, ValidationInfo, field_validator


class UserModel(BaseModel):
    password: str
    password_repeat: str

    @field_validator('password_repeat', mode='after')
    @classmethod
    def check_passwords_match(cls, value: str, info: ValidationInfo) -> str:
        if value != info.data['password']:
            raise ValueError('Passwords do not match')
        return value
```

```python
class Model(BaseModel):
    text: str

    @field_validator('text', mode='after')
    @classmethod
    def remove_stopwords(cls, v: str, info: ValidationInfo) -> str:
        if isinstance(info.context, dict):
            stopwords = info.context.get('stopwords', set())
            v = ' '.join(w for w in v.split() if w.lower() not in stopwords)
        return v


print(Model.model_validate(
    {'text': 'This is an example document'},
    context={'stopwords': ['this', 'is', 'an']},
))
#> text='example document'
```

## Annotated ordering

Before/wrap run **right-to-left**; after run **left-to-right**. Decorator-defined validators are appended after annotated ones, then ordered by the same rule.

```python
class Model(BaseModel):
    name: Annotated[
        str,
        AfterValidator(runs_3rd),
        AfterValidator(runs_4th),
        BeforeValidator(runs_2nd),
        WrapValidator(runs_1st),
    ]
```

## Special utilities

- `InstanceOf[T]` — value must be an instance of `T`.
- `SkipValidation[T]` — bypass validation entirely.
- `PydanticUseDefault` — raise from a `BeforeValidator` to fall back to the field default.

For before/plain/wrap validators that accept wider inputs than the field type, set `json_schema_input_type=...` on `@field_validator` so JSON schema reflects the real input.

See [[Pydantic CO 02 - Fields]], [[Pydantic CO 09 - Serialization]], [[Pydantic CO 13 - Strict Mode]].

💡 **Takeaway:** Default to `mode='after'` annotated validators packaged as named `Annotated` aliases; reach for `before`/`wrap` only when you need to massage raw input or recover from Pydantic's errors.
