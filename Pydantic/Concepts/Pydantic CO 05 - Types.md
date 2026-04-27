# CO 05 — Types

🔑 **Key insight:** Pydantic accepts almost any Python type out of the box — and for the rest, the `Annotated` pattern + `__get_pydantic_core_schema__` lets you bolt validation, serialization, and JSON Schema onto anything.

## Built-in support

Stdlib and primitive types just work — `int`, `float`, `bool`, `str`, `bytes`, `Decimal`, `datetime`, `date`, `time`, `timedelta`, `UUID`, `Path`, `Enum`, `Literal`, `Pattern`, IP addresses, `set`/`frozenset`, `list`/`tuple`/`dict`, `Annotated`. Coercion rules per source mode (Python vs JSON vs strings) are catalogued in [[Pydantic CO 16 - Conversion Table]].

Pydantic-specific types live in `pydantic` and `pydantic-extra-types`:

- Strict variants: `StrictInt`, `StrictStr`, `StrictBool`, `StrictFloat`, `StrictBytes` — see [[Pydantic CO 13 - Strict Mode]].
- Constrained: `PositiveInt`, `NegativeInt`, `NonNegativeInt`, `NonPositiveInt`, `PositiveFloat`, etc.
- Strings: `EmailStr`, `NameEmail`, `SecretStr`, `SecretBytes`.
- URLs: `AnyUrl`, `HttpUrl`, `PostgresDsn`, `RedisDsn`, …
- Files: `FilePath`, `DirectoryPath`.
- Misc: `Json`, `Color` (in extras), `PaymentCardNumber` (in extras).

## The annotated pattern

Bolt a constraint onto an existing type without subclassing:

```python
from typing import Annotated
from pydantic import Field, TypeAdapter, ValidationError

PositiveInt = Annotated[int, Field(gt=0)]

ta = TypeAdapter(PositiveInt)
print(ta.validate_python(1))
#> 1

try:
    ta.validate_python(-1)
except ValidationError as exc:
    print(exc)
    """
    1 validation error for constrained-int
      Input should be greater than 0 [type=greater_than, input_value=-1, input_type=int]
    """
```

Or use `annotated-types` directly:

```python
from annotated_types import Gt
PositiveInt = Annotated[int, Gt(0)]
```

### Stack validators, serializers, and schema overrides

```python
from typing import Annotated
from pydantic import (
    AfterValidator, PlainSerializer, TypeAdapter, WithJsonSchema,
)

TruncatedFloat = Annotated[
    float,
    AfterValidator(lambda x: round(x, 1)),
    PlainSerializer(lambda x: f'{x:.1e}', return_type=str),
    WithJsonSchema({'type': 'string'}, mode='serialization'),
]

ta = TypeAdapter(TruncatedFloat)

input = 1.02345
assert ta.validate_python(input) == 1.0
assert ta.dump_json(input) == b'"1.0e+00"'
assert ta.json_schema(mode='validation') == {'type': 'number'}
assert ta.json_schema(mode='serialization') == {'type': 'string'}
```

### Generic annotated types

```python
from typing import Annotated, TypeVar
from annotated_types import Gt, Len
from pydantic import TypeAdapter, ValidationError

T = TypeVar('T')

ShortList = Annotated[list[T], Len(max_length=4)]

ta = TypeAdapter(ShortList[int])
v = ta.validate_python([1, 2, 3, 4])
assert v == [1, 2, 3, 4]

PositiveList = list[Annotated[T, Gt(0)]]
ta = TypeAdapter(PositiveList[float])
assert type(ta.validate_python([1.0])[0]) is float
```

## Named type aliases (v2.11+)

Reusable, schema-deduplicated aliases. Python 3.9+:

```python
from typing import Annotated
from annotated_types import Gt
from typing_extensions import TypeAliasType
from pydantic import BaseModel

PositiveIntList = TypeAliasType('PositiveIntList', list[Annotated[int, Gt(0)]])

class Model(BaseModel):
    x: PositiveIntList
    y: PositiveIntList
```

Python 3.12+:

```python
from typing import Annotated
from annotated_types import Gt
from pydantic import BaseModel

type PositiveIntList = list[Annotated[int, Gt(0)]]

class Model(BaseModel):
    x: PositiveIntList
    y: PositiveIntList
```

Both produce a single `$defs.PositiveIntList` referenced by `x` and `y` — no inline duplication. ⚠️ Field-specific metadata (`alias`, `default`, `deprecated`) is not honored inside named aliases.

### Recursive named aliases

```python
from typing import Union
from typing_extensions import TypeAliasType
from pydantic import TypeAdapter

Json = TypeAliasType(
    'Json',
    'Union[dict[str, Json], list[Json], str, int, float, bool, None]',
)

ta = TypeAdapter(Json)
```

Python 3.12+:

```python
type Json = dict[str, Json] | list[Json] | str | int | float | bool | None
```

## Custom types via `__get_pydantic_core_schema__`

### As a classmethod on the type

```python
from typing import Any
from pydantic_core import CoreSchema, core_schema
from pydantic import GetCoreSchemaHandler, TypeAdapter

class Username(str):
    @classmethod
    def __get_pydantic_core_schema__(
        cls, source_type: Any, handler: GetCoreSchemaHandler
    ) -> CoreSchema:
        return core_schema.no_info_after_validator_function(cls, handler(str))

ta = TypeAdapter(Username)
res = ta.validate_python('abc')
assert isinstance(res, Username)
```

### As an Annotated marker (preferred for third-party types)

```python
from dataclasses import dataclass
from typing import Annotated, Any, Callable
from pydantic_core import CoreSchema, core_schema
from pydantic import BaseModel, GetCoreSchemaHandler

@dataclass(frozen=True)
class MyAfterValidator:
    func: Callable[[Any], Any]

    def __get_pydantic_core_schema__(
        self, source_type: Any, handler: GetCoreSchemaHandler
    ) -> CoreSchema:
        return core_schema.no_info_after_validator_function(
            self.func, handler(source_type)
        )

Username = Annotated[str, MyAfterValidator(str.lower)]

class Model(BaseModel):
    name: Username

assert Model(name='ABC').name == 'abc'
```

### Wrapping a third-party type

```python
from typing import Annotated, Any
from pydantic_core import core_schema
from pydantic import BaseModel, GetCoreSchemaHandler, GetJsonSchemaHandler
from pydantic.json_schema import JsonSchemaValue

class ThirdPartyType:
    x: int
    def __init__(self):
        self.x = 0

class _ThirdPartyTypePydanticAnnotation:
    @classmethod
    def __get_pydantic_core_schema__(cls, _source_type, _handler):
        def validate_from_int(value: int) -> ThirdPartyType:
            result = ThirdPartyType()
            result.x = value
            return result

        from_int_schema = core_schema.chain_schema([
            core_schema.int_schema(),
            core_schema.no_info_plain_validator_function(validate_from_int),
        ])

        return core_schema.json_or_python_schema(
            json_schema=from_int_schema,
            python_schema=core_schema.union_schema([
                core_schema.is_instance_schema(ThirdPartyType),
                from_int_schema,
            ]),
            serialization=core_schema.plain_serializer_function_ser_schema(
                lambda instance: instance.x
            ),
        )

    @classmethod
    def __get_pydantic_json_schema__(cls, _core_schema, handler):
        return handler(core_schema.int_schema())

PydanticThirdPartyType = Annotated[ThirdPartyType, _ThirdPartyTypePydanticAnnotation]

class Model(BaseModel):
    third_party_type: PydanticThirdPartyType
```

`json_or_python_schema` lets the same field accept JSON-form input (here: an int) and Python-form input (an instance) with different validation paths.

### `GetPydanticSchema` — minimal boilerplate

```python
from typing import Annotated
from pydantic_core import core_schema
from pydantic import BaseModel, GetPydanticSchema

class Model(BaseModel):
    y: Annotated[
        str,
        GetPydanticSchema(
            lambda tp, handler: core_schema.no_info_after_validator_function(
                lambda x: x * 2, handler(tp)
            )
        ),
    ]

assert Model(y='ab').y == 'abab'
```

## Custom generic classes

```python
from dataclasses import dataclass
from typing import Any, Generic, TypeVar
from pydantic_core import CoreSchema, core_schema
from typing_extensions import get_args, get_origin
from pydantic import BaseModel, GetCoreSchemaHandler, ValidatorFunctionWrapHandler

ItemType = TypeVar('ItemType')

@dataclass
class Owner(Generic[ItemType]):
    name: str
    item: ItemType

    @classmethod
    def __get_pydantic_core_schema__(
        cls, source_type: Any, handler: GetCoreSchemaHandler
    ) -> CoreSchema:
        origin = get_origin(source_type)
        if origin is None:
            origin = source_type
            item_tp = Any
        else:
            item_tp = get_args(source_type)[0]
        item_schema = handler.generate_schema(item_tp)

        def val_item(v: Owner[Any], handler: ValidatorFunctionWrapHandler) -> Owner[Any]:
            v.item = handler(v.item)
            return v

        python_schema = core_schema.chain_schema([
            core_schema.is_instance_schema(cls),
            core_schema.no_info_wrap_validator_function(val_item, item_schema),
        ])

        return core_schema.json_or_python_schema(
            json_schema=core_schema.chain_schema([
                core_schema.typed_dict_schema({
                    'name': core_schema.typed_dict_field(core_schema.str_schema()),
                    'item': core_schema.typed_dict_field(item_schema),
                }),
                core_schema.no_info_before_validator_function(
                    lambda data: Owner(name=data['name'], item=data['item']),
                    python_schema,
                ),
            ]),
            python_schema=python_schema,
        )
```

Use `get_args(source_type)` to extract type parameters, then ask `handler.generate_schema()` to build their schemas.

## Field-name access (v2.4+)

Custom validators can know which field they're validating:

```python
from typing import Any
from pydantic_core import core_schema
from pydantic import BaseModel, GetCoreSchemaHandler, ValidationInfo

class CustomType:
    def __init__(self, value: int, field_name: str):
        self.value = value
        self.field_name = field_name

    def __repr__(self):
        return f'CustomType<{self.value} {self.field_name!r}>'

    @classmethod
    def validate(cls, value: int, info: ValidationInfo):
        return cls(value, info.field_name)

    @classmethod
    def __get_pydantic_core_schema__(cls, source_type, handler):
        return core_schema.with_info_after_validator_function(
            cls.validate, handler(int)
        )

class MyModel(BaseModel):
    my_field: CustomType

print(MyModel(my_field=1).my_field)
#> CustomType<1 'my_field'>
```

Or via an `AfterValidator` marker:

```python
from typing import Annotated
from pydantic import AfterValidator, BaseModel, ValidationInfo

def my_validators(value: int, info: ValidationInfo):
    return f'<{value} {info.field_name!r}>'

class MyModel(BaseModel):
    my_field: Annotated[int, AfterValidator(my_validators)]
```

## Custom-type decision tree

1. **Need a constraint?** Use `Field(gt=..., max_length=..., ...)` or annotated-types markers.
2. **Need a validator/serializer?** Use `Annotated[T, AfterValidator(...), PlainSerializer(...)]`.
3. **Wrapping a third-party class you don't control?** Annotated marker class with `__get_pydantic_core_schema__`.
4. **Owning the class?** `__get_pydantic_core_schema__` as a classmethod on the type itself.
5. **Schema-only override?** `WithJsonSchema(...)` or `__get_pydantic_json_schema__` — see [[Pydantic CO 03 - JSON Schema]].

💡 **Takeaway:** `Annotated[...]` is the ladder — start with markers, climb to `__get_pydantic_core_schema__` only when you need full control over validation, serialization, and schema in one shot.
