# EX 05 — Custom Validators

🔑 **Key insight:** when a constraint cannot be expressed as a type, attach it as `Annotated` metadata implementing `__get_pydantic_core_schema__` — for nested cross-field rules, lift the check to the parent with `@model_validator` or thread state through validation `context`.

## Reusable validators via `Annotated` metadata

For constraints you want to apply to many fields (timezone-aware datetimes, UTC offset bounds, etc.), build a dataclass that implements `__get_pydantic_core_schema__` and wraps a core-schema validator. Below: a datetime validator that asserts the timezone matches an expected name.

```python
import datetime as dt
from dataclasses import dataclass
from pprint import pprint
from typing import Annotated, Any, Callable, Optional

import pytz
from pydantic_core import CoreSchema, core_schema

from pydantic import (
  GetCoreSchemaHandler,
  PydanticUserError,
  TypeAdapter,
  ValidationError,
)


@dataclass(frozen=True)
class MyDatetimeValidator:
  tz_constraint: Optional[str] = None

  def tz_constraint_validator(
      self,
      value: dt.datetime,
      handler: Callable,  # (1)
  ):
      """Validate tz_constraint and tz_info."""
      # handle naive datetimes
      if self.tz_constraint is None:
          assert (
              value.tzinfo is None
          ), 'tz_constraint is None, but provided value is tz-aware.'
          return handler(value)

      # validate tz_constraint and tz-aware tzinfo
      if self.tz_constraint not in pytz.all_timezones:
          raise PydanticUserError(
              f'Invalid tz_constraint: {self.tz_constraint}',
              code='unevaluable-type-annotation',
          )
      result = handler(value)  # (2)
      assert self.tz_constraint == str(
          result.tzinfo
      ), f'Invalid tzinfo: {str(result.tzinfo)}, expected: {self.tz_constraint}'

      return result

  def __get_pydantic_core_schema__(
      self,
      source_type: Any,
      handler: GetCoreSchemaHandler,
  ) -> CoreSchema:
      return core_schema.no_info_wrap_validator_function(
          self.tz_constraint_validator,
          handler(source_type),
      )


LA = 'America/Los_Angeles'
ta = TypeAdapter(Annotated[dt.datetime, MyDatetimeValidator(LA)])
print(
  ta.validate_python(dt.datetime(2023, 1, 1, 0, 0, tzinfo=pytz.timezone(LA)))
)
#> 2023-01-01 00:00:00-07:53

LONDON = 'Europe/London'
try:
  ta.validate_python(
      dt.datetime(2023, 1, 1, 0, 0, tzinfo=pytz.timezone(LONDON))
  )
except ValidationError as ve:
  pprint(ve.errors(), width=100)
  """
  [{'ctx': {'error': AssertionError('Invalid tzinfo: Europe/London, expected: America/Los_Angeles')},
  'input': datetime.datetime(2023, 1, 1, 0, 0, tzinfo=<DstTzInfo 'Europe/London' LMT-1 day, 23:59:00 STD>),
  'loc': (),
  'msg': 'Assertion failed, Invalid tzinfo: Europe/London, expected: America/Los_Angeles',
  'type': 'assertion_error',
  'url': 'https://errors.pydantic.dev/2.8/v/assertion_error'}]
  """
```

`no_info_wrap_validator_function` is the *wrap* validator — it runs your code, decides whether to call `handler(value)` (which executes the inner Pydantic validation), and returns the final value. That gives you both pre- and post-validation hooks in one place. See [[Pydantic CO 10 - Validators]] for the full validator taxonomy and [[Pydantic CO 14 - Type Adapter]] for `TypeAdapter`.

## Parameterized constraint: UTC offset bounds

Same shape, different rule — assert the offset falls inside `[lower_bound, upper_bound]` hours:

```python
import datetime as dt
from dataclasses import dataclass
from pprint import pprint
from typing import Annotated, Any, Callable

import pytz
from pydantic_core import CoreSchema, core_schema

from pydantic import GetCoreSchemaHandler, TypeAdapter, ValidationError


@dataclass(frozen=True)
class MyDatetimeValidator:
    lower_bound: int
    upper_bound: int

    def validate_tz_bounds(self, value: dt.datetime, handler: Callable):
        """Validate and test bounds"""
        assert value.utcoffset() is not None, 'UTC offset must exist'
        assert self.lower_bound <= self.upper_bound, 'Invalid bounds'

        result = handler(value)

        hours_offset = value.utcoffset().total_seconds() / 3600
        assert (
            self.lower_bound <= hours_offset <= self.upper_bound
        ), 'Value out of bounds'

        return result

    def __get_pydantic_core_schema__(
        self,
        source_type: Any,
        handler: GetCoreSchemaHandler,
    ) -> CoreSchema:
        return core_schema.no_info_wrap_validator_function(
            self.validate_tz_bounds,
            handler(source_type),
        )


LA = 'America/Los_Angeles'  # UTC-7 or UTC-8
ta = TypeAdapter(Annotated[dt.datetime, MyDatetimeValidator(-10, -5)])
print(
    ta.validate_python(dt.datetime(2023, 1, 1, 0, 0, tzinfo=pytz.timezone(LA)))
)
#> 2023-01-01 00:00:00-07:53

LONDON = 'Europe/London'
try:
    print(
        ta.validate_python(
            dt.datetime(2023, 1, 1, 0, 0, tzinfo=pytz.timezone(LONDON))
        )
    )
except ValidationError as e:
    pprint(e.errors(), width=100)
    """
    [{'ctx': {'error': AssertionError('Value out of bounds')},
    'input': datetime.datetime(2023, 1, 1, 0, 0, tzinfo=<DstTzInfo 'Europe/London' LMT-1 day, 23:59:00 STD>),
    'loc': (),
    'msg': 'Assertion failed, Value out of bounds',
    'type': 'assertion_error',
    'url': 'https://errors.pydantic.dev/2.8/v/assertion_error'}]
    """
```

## Cross-field rules — validate at the parent

When a child needs to know about its sibling (e.g. "user password must not appear in the parent's `forbidden_passwords`"), the cleanest move is `@model_validator(mode='after')` on the parent:

```python
from typing_extensions import Self

from pydantic import BaseModel, ValidationError, model_validator


class User(BaseModel):
    username: str
    password: str


class Organization(BaseModel):
    forbidden_passwords: list[str]
    users: list[User]

    @model_validator(mode='after')
    def validate_user_passwords(self) -> Self:
        """Check that user password is not in forbidden list. Raise a validation error if a forbidden password is encountered."""
        for user in self.users:
            current_pw = user.password
            if current_pw in self.forbidden_passwords:
                raise ValueError(
                    f'Password {current_pw} is forbidden. Please choose another password for user {user.username}.'
                )
        return self


data = {
    'forbidden_passwords': ['123'],
    'users': [
        {'username': 'Spartacat', 'password': '123'},
        {'username': 'Iceburgh', 'password': '87'},
    ],
}
try:
    org = Organization(**data)
except ValidationError as e:
    print(e)
    """
    1 validation error for Organization
      Value error, Password 123 is forbidden. Please choose another password for user Spartacat. [type=value_error, input_value={'forbidden_passwords': [...gh', 'password': '87'}]}, input_type=dict]
    """
```

## Cross-field rules — push state down via `context`

If you need the error pinpointed at the offending child field (`users.0.password`) rather than the parent, use a `@field_validator` on the child and seed the validation `context` from the parent:

```python
from pydantic import BaseModel, ValidationError, ValidationInfo, field_validator


class User(BaseModel):
    username: str
    password: str

    @field_validator('password', mode='after')
    @classmethod
    def validate_user_passwords(
        cls, password: str, info: ValidationInfo
    ) -> str:
        """Check that user password is not in forbidden list."""
        forbidden_passwords = (
            info.context.get('forbidden_passwords', []) if info.context else []
        )
        if password in forbidden_passwords:
            raise ValueError(f'Password {password} is forbidden.')
        return password


class Organization(BaseModel):
    forbidden_passwords: list[str]
    users: list[User]

    @field_validator('forbidden_passwords', mode='after')
    @classmethod
    def add_context(cls, v: list[str], info: ValidationInfo) -> list[str]:
        if info.context is not None:
            info.context.update({'forbidden_passwords': v})
        return v


data = {
    'forbidden_passwords': ['123'],
    'users': [
        {'username': 'Spartacat', 'password': '123'},
        {'username': 'Iceburgh', 'password': '87'},
    ],
}

try:
    org = Organization.model_validate(data, context={})
except ValidationError as e:
    print(e)
    """
    1 validation error for Organization
    users.0.password
      Value error, Password 123 is forbidden. [type=value_error, input_value='123', input_type=str]
    """
```

⚠️ The caller **must** pass `context={}` to `model_validate` — `Organization(**data)` won't work here. If you forget, `info.context` is `None` and the check silently no-ops. The docs warn: "The ability to mutate the context within a validator adds a lot of power to nested validation, but can also lead to confusing or hard-to-debug code."

For the resulting `ValidationError` shape, see [[Pydantic ER 02 - Validation Errors]].

💡 **Takeaway:** reusable type-level rules go in `Annotated` metadata; cross-field rules live on the parent (`@model_validator`) or thread state through `context` when you need precise error locations.
