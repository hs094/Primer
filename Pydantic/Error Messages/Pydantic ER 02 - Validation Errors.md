# ER 02 — Validation Errors

🔑 **Key insight:** Every `ValidationError` entry carries a stable `type` string drawn from a fixed catalogue — branch on the code, not on the human-readable `msg`, when you need programmatic reactions.

The codes below are surfaced through `error['type']` in [[Pydantic ER 01 - Error Handling|`ValidationError.errors()`]]. They are produced by `pydantic-core` (see [[Pydantic IT 01 - Architecture]]).

## Type-coercion failures

| Code | Trigger |
|---|---|
| `int_parsing` | String unparseable as int — `Model(x='test')` for an int field |
| `int_parsing_size` | String too long to parse as int (e.g. 4 301 `'1'` chars) |
| `int_from_float` | Float provided for int field — `Model(x=0.5)` |
| `int_type` | Wrong input type for int — `Model(x=None)` |
| `float_parsing` | String unparseable as float |
| `float_type` | Wrong input type for float field |
| `bool_parsing` | String input cannot coerce to bool |
| `bool_type` | Wrong input type for bool — `Model(x=None)` |
| `complex_str_parsing` | `"abc"` for complex field |
| `complex_type` | Cannot interpret as complex |
| `string_type` | Wrong input type for str — `Model(x=1)` |
| `string_unicode` | Unparseable as Unicode — `Model(x=b'\x81')` |
| `string_sub_type` | Strict str field receives a str subtype (e.g. an enum value) |
| `string_not_ascii` | Non-ASCII when `ascii_only=True` |
| `bytes_type` | Wrong input type for bytes |
| `bytes_invalid_encoding` | Invalid under configured encoding (e.g. odd hex digits) |

## Length / range / pattern constraints

| Code | Trigger |
|---|---|
| `string_too_short` / `string_too_long` | violates `min_length` / `max_length` on str |
| `bytes_too_short` / `bytes_too_long` | violates length on bytes |
| `too_short` / `too_long` | violates length on collections (list, set, tuple, …) |
| `string_pattern_mismatch` | doesn't match `pattern` regex |
| `greater_than` / `greater_than_equal` | violates `gt` / `ge` |
| `less_than` / `less_than_equal` | violates `lt` / `le` |
| `multiple_of` | not a multiple of `multiple_of` |
| `finite_number` | infinite or out-of-range float for an int field |

## Numeric / decimal specifics

| Code | Trigger |
|---|---|
| `decimal_parsing` | String unparseable as `Decimal` |
| `decimal_type` | Wrong input type for `Decimal` |
| `decimal_max_digits` | Too many total digits |
| `decimal_max_places` | Too many decimal places |
| `decimal_whole_digits` | Too many whole digits given `max_digits` + `decimal_places` |

## Date / time

| Code | Trigger |
|---|---|
| `date_type` / `datetime_type` / `time_type` / `time_delta_type` | wrong input type |
| `date_parsing` / `datetime_parsing` / `time_parsing` / `time_delta_parsing` | unparseable string |
| `datetime_from_date_parsing` | invalid datetime string (e.g. `'2023-13-01'`) |
| `date_from_datetime_inexact` | datetime has non-zero time when fed to a date field |
| `date_future` / `date_past` | `FutureDate` / `PastDate` violated |
| `datetime_future` / `datetime_past` | `FutureDatetime` / `PastDatetime` violated |
| `datetime_object_invalid` | Datetime object itself broken (e.g. bad `tzinfo`) |
| `timezone_aware` / `timezone_naive` | wrong tz-awareness for `AwareDatetime` / `NaiveDatetime` |

## Container types

| Code | Trigger |
|---|---|
| `list_type` / `set_type` / `frozen_set_type` / `tuple_type` / `dict_type` | wrong input type for the container |
| `iterable_type` / `iteration_error` | invalid iterable / generator raised mid-iteration |
| `mapping_type` | broken `Mapping` protocol (e.g. `.items()` raises) |
| `set_item_not_hashable` | unhashable value placed in a set/frozenset |
| `invalid_key` | dict key isn't a string instance — e.g. `b'y'` key |
| `recursion_loop` | cyclic reference detected during validation |

## Model & dataclass shape

| Code | Trigger |
|---|---|
| `missing` | Required field absent from input |
| `extra_forbidden` | Extra fields with `extra='forbid'` (see [[Pydantic CO 08 - Configuration]]) |
| `frozen_field` | Assigning/deleting a `frozen=True` field |
| `frozen_instance` | Mutating any field on a frozen model |
| `no_such_attribute` | Setting unknown attr with `validate_assignment=True` |
| `model_type` | Input to a model isn't dict/instance — `Model.model_validate('x')` |
| `model_attributes_type` | Input not dict / model / valid object under `from_attributes=True` |
| `dataclass_type` / `dataclass_exact_type` | dataclass input type wrong (esp. with [[Pydantic CO 13 - Strict Mode|strict]]) |
| `is_instance_of` / `is_subclass_of` | failed `InstanceOf` / `SubclassOf` check |
| `none_required` | Non-None value for `None`-typed field |
| `default_factory_not_called` | factory needs prior fields that themselves failed |
| `get_attribute_error` | `from_attributes=True` and a property raised |
| `missing_sentinel_error` | experimental `MISSING` sentinel not provided |

## Callable / function validation

| Code | Trigger |
|---|---|
| `arguments_type` | Object passed to function validation isn't tuple/list/dict |
| `callable_type` | Input invalid as `Callable` (e.g. bad `ImportString`) |
| `missing_argument` | required positional-or-keyword arg missing |
| `missing_keyword_only_argument` | required `*, a:` arg missing |
| `missing_positional_only_argument` | required `a:int, /` arg missing |
| `multiple_argument_values` | same arg passed twice — `foo(1, a=2)` |
| `unexpected_keyword_argument` | kwarg given to positional-only param |
| `unexpected_positional_argument` | positional given to keyword-only param |

## Unions, literals, enums (see [[Pydantic CO 06 - Unions]])

| Code | Trigger |
|---|---|
| `literal_error` | Input not in `Literal[...]` values |
| `enum` | Input not a member of the Enum |
| `union_tag_invalid` | Discriminator value not one of the expected tags |
| `union_tag_not_found` | Discriminator key absent from input |

## URLs / UUID / JSON

| Code | Trigger |
|---|---|
| `url_parsing` / `url_syntax_violation` / `url_scheme` / `url_too_long` / `url_type` | bad URL input / wrong scheme / >2083 chars |
| `uuid_parsing` / `uuid_type` / `uuid_version` | invalid UUID string / type / wrong version |
| `json_invalid` | Not valid JSON for a `Json[...]` field |
| `json_type` | Input type cannot be parsed as JSON |
| `needs_python_object` | Tried to validate a `type[BaseModel]` field from JSON |

## Custom-validator escape hatches (see [[Pydantic CO 10 - Validators]])

| Code | Trigger |
|---|---|
| `value_error` | a `@field_validator` / `@model_validator` raised `ValueError` |
| `assertion_error` | an `assert` inside a validator failed |

⚠️ Match on `error['type']`, never on `error['msg']` — messages are localised/cosmetic; codes are the contract.

💡 **Takeaway:** Treat this catalogue as the public enum of pydantic-core — your error-mapping layer should switch on these strings and translate them into your own domain or HTTP problem details.
