# CO 03 — JSON Schema

🔑 **Key insight:** `model_json_schema()` produces Draft 2020-12 / OpenAPI 3.1 JSON Schema for free — and every layer (Field, Annotated, model config, custom generator) has a hook for customizing it.

## Generating

```python
import json
from enum import Enum
from typing import Annotated, Union
from pydantic import BaseModel, Field
from pydantic.config import ConfigDict

class FooBar(BaseModel):
    count: int
    size: Union[float, None] = None

class Gender(str, Enum):
    male = 'male'; female = 'female'; other = 'other'; not_given = 'not_given'

class MainModel(BaseModel):
    """This is the description of the main model"""
    model_config = ConfigDict(title='Main')

    foo_bar: FooBar
    gender: Annotated[Union[Gender, None], Field(alias='Gender')] = None
    snap: int = Field(
        default=42, title='The Snap',
        description='this is the value of snap', gt=30, lt=50,
    )

print(json.dumps(MainModel.model_json_schema(), indent=2))
```

For arbitrary types use [[Pydantic CO 14 - Type Adapter]]:

```python
from pydantic import TypeAdapter
adapter = TypeAdapter(list[int])
print(adapter.json_schema())
#> {'items': {'type': 'integer'}, 'type': 'array'}
```

⚠️ Don't confuse `model_json_schema()` (returns the schema) with `model_dump_json()` (serializes an instance — see [[Pydantic CO 04 - JSON]]).

## Validation vs serialization mode

```python
from decimal import Decimal
from pydantic import BaseModel

class Model(BaseModel):
    a: Decimal = Decimal('12.34')

print(Model.model_json_schema(mode='validation'))
# a accepts {type: number} OR {type: string, pattern: ...}

print(Model.model_json_schema(mode='serialization'))
# a is {type: string, pattern: ...} only
```

Validation schema accepts what comes in; serialization schema describes what goes out. They diverge for types like `Decimal`, `SecretStr`, `bytes`.

## Field-level customization

```python
from typing import Annotated
from pydantic import BaseModel, EmailStr, Field, SecretStr

class User(BaseModel):
    age: int = Field(description='Age of the user')
    email: Annotated[EmailStr, Field(examples=['[email protected]'])]
    name: str = Field(title='Username')
    password: SecretStr = Field(
        json_schema_extra={
            'title': 'Password',
            'description': 'Password of the user',
            'examples': ['123456'],
        }
    )
```

`title`, `description`, `examples`, `json_schema_extra` all surface in the schema.

### `json_schema_extra` as callable (mutate in place)

```python
from pydantic import BaseModel, Field

def pop_default(s):
    s.pop('default')

class Model(BaseModel):
    a: int = Field(default=1, json_schema_extra=pop_default)
```

### Merging (v2.9+)

```python
from typing import Annotated
from typing_extensions import TypeAlias
from pydantic import Field, TypeAdapter

ExternalType: TypeAlias = Annotated[int, Field(json_schema_extra={'key1': 'value1'})]
ta = TypeAdapter(Annotated[ExternalType, Field(json_schema_extra={'key2': 'value2'})])
print(ta.json_schema())
#> {'key1': 'value1', 'key2': 'value2', 'type': 'integer'}
```

Dict-form `json_schema_extra` from nested `Annotated` layers merges additively.

## Model-level customization

```python
from pydantic import BaseModel, ConfigDict

class Model(BaseModel):
    a: str
    model_config = ConfigDict(json_schema_extra={'examples': [{'a': 'Foo'}]})
```

`ConfigDict` knobs: `title`, `json_schema_extra`, `field_title_generator`, `model_title_generator`, `json_schema_mode_override`. See [[Pydantic CO 08 - Configuration]].

## Title generators

```python
from pydantic import BaseModel, Field
from pydantic.fields import FieldInfo

def make_title(field_name: str, field_info: FieldInfo) -> str:
    return field_name.upper()

class Person(BaseModel):
    name: str = Field(field_title_generator=make_title)
    age: int = Field(field_title_generator=make_title)
```

Model-wide:

```python
from pydantic import BaseModel, ConfigDict

class Person(BaseModel):
    model_config = ConfigDict(
        field_title_generator=lambda fn, fi: fn.upper()
    )
    name: str
    age: int
```

```python
def make_title(model: type) -> str:
    return f'Title-{model.__name__}'

class Person(BaseModel):
    model_config = ConfigDict(model_title_generator=make_title)
    name: str
    age: int
```

## `WithJsonSchema` — override for a type

```python
from typing import Annotated
from pydantic import BaseModel, WithJsonSchema

MyInt = Annotated[int, WithJsonSchema({'type': 'integer', 'examples': [1, 0, -1]})]

class Model(BaseModel):
    a: MyInt
```

Preferred over `__get_pydantic_json_schema__` when you just want to swap the schema for a type.

## `SkipJsonSchema` — drop a field/branch

`Annotated[T, SkipJsonSchema()]` excludes the field (or union branch) from the generated schema while keeping it for validation.

## `__get_pydantic_json_schema__` — modify only the JSON Schema

```python
import json
from typing import Any
from pydantic_core import core_schema as cs
from pydantic import GetCoreSchemaHandler, GetJsonSchemaHandler, TypeAdapter
from pydantic.json_schema import JsonSchemaValue

class Person:
    def __init__(self, name: str, age: int):
        self.name = name; self.age = age

    @classmethod
    def __get_pydantic_core_schema__(cls, source_type: Any, handler: GetCoreSchemaHandler) -> cs.CoreSchema:
        return cs.typed_dict_schema({
            'name': cs.typed_dict_field(cs.str_schema()),
            'age': cs.typed_dict_field(cs.int_schema()),
        })

    @classmethod
    def __get_pydantic_json_schema__(cls, core_schema: cs.CoreSchema, handler: GetJsonSchemaHandler) -> JsonSchemaValue:
        json_schema = handler(core_schema)
        json_schema = handler.resolve_ref_schema(json_schema)
        json_schema['examples'] = [{'name': 'John Doe', 'age': 25}]
        json_schema['title'] = 'Person'
        return json_schema
```

This affects only the JSON Schema output — not validation/serialization. For full control, use `__get_pydantic_core_schema__` (see [[Pydantic CO 05 - Types]]).

## Top-level multi-model schema

```python
from pydantic.json_schema import models_json_schema

_, top = models_json_schema(
    [(Model, 'validation'), (Bar, 'validation')], title='My Schema'
)
```

Bundles several models under one `$defs`.

## Custom generator — `GenerateJsonSchema`

```python
from pydantic import BaseModel
from pydantic.json_schema import GenerateJsonSchema

class MyGenerateJsonSchema(GenerateJsonSchema):
    def generate(self, schema, mode='validation'):
        json_schema = super().generate(schema, mode=mode)
        json_schema['title'] = 'Customize title'
        json_schema['$schema'] = self.schema_dialect
        return json_schema

class MyModel(BaseModel):
    x: int

print(MyModel.model_json_schema(schema_generator=MyGenerateJsonSchema))
```

Skip un-schema-able fields:

```python
from typing import Callable
from pydantic_core import PydanticOmit, core_schema
from pydantic.json_schema import GenerateJsonSchema, JsonSchemaValue

class MyGenerateJsonSchema(GenerateJsonSchema):
    def handle_invalid_for_json_schema(self, schema, error_info):
        raise PydanticOmit
```

Override `sort()` to control key ordering — Pydantic alphabetizes everywhere except inside `properties`.

## `ref_template`

```python
adapter.json_schema(ref_template='#/components/schemas/{model}')
```

Definitions still live under `$defs`, but `$ref` strings use your prefix — handy for OpenAPI integration.

💡 **Takeaway:** Start with `Field(...)` metadata; reach for `WithJsonSchema` for a type swap; only subclass `GenerateJsonSchema` when you need cross-cutting changes.
