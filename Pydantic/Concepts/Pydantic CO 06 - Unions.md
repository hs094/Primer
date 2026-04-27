# CO 06 — Unions

🔑 **Key insight:** Unions only need *one* member to validate; how Pydantic picks which member is the entire game — and a discriminator beats both heuristics.

## Three modes

1. **Smart mode** (default in v2). Tries members, scores by exactness + valid-fields-set, picks best.
2. **Left-to-right.** Tries members in order, accepts the first that succeeds.
3. **Discriminated.** A tag in the input picks the member directly — no trial and error.

## Smart mode (default)

Scoring buckets, highest first: exact type match → strict-mode pass → lax-mode pass. For models/dataclasses/TypedDicts, "valid fields set" count breaks ties.

```python
from typing import Union
from uuid import UUID

from pydantic import BaseModel


class User(BaseModel):
    id: Union[int, str, UUID]
    name: str


print(User(id=123, name='John Doe').id)
#> 123
print(User(id='1234', name='John Doe').id)
#> 1234
u = UUID('cf57432e-809e-4353-adbd-9d5c0d733868')
print(User(id=u, name='John Doe').id)
#> cf57432e-809e-4353-adbd-9d5c0d733868
```

## Left-to-right mode

Set per-field via `Field(union_mode='left_to_right')`:

```python
from typing import Union

from pydantic import BaseModel, Field


class User(BaseModel):
    id: Union[str, int] = Field(union_mode='left_to_right')
```

⚠️ Order matters. With `Union[int, str]` in left-to-right mode, the lax string `'456'` is *accepted as int first*, so you get `id=456` (int), not `'456'` (str).

## Discriminated unions (tagged unions)

Faster, cleaner errors, OpenAPI `discriminator` in JSON schema. Use a `Literal` tag field:

```python
from typing import Literal, Union

from pydantic import BaseModel, Field, ValidationError


class Cat(BaseModel):
    pet_type: Literal['cat']
    meows: int


class Dog(BaseModel):
    pet_type: Literal['dog']
    barks: float


class Lizard(BaseModel):
    pet_type: Literal['reptile', 'lizard']
    scales: bool


class Model(BaseModel):
    pet: Union[Cat, Dog, Lizard] = Field(discriminator='pet_type')
    n: int


print(Model(pet={'pet_type': 'dog', 'barks': 3.14}, n=1))
#> pet=Dog(pet_type='dog', barks=3.14) n=1
try:
    Model(pet={'pet_type': 'dog'}, n=1)
except ValidationError as e:
    print(e)
    """
    1 validation error for Model
    pet.dog.barks
      Field required [type=missing, input_value={'pet_type': 'dog'}, input_type=dict]
    """
```

## Callable discriminator

When the tag isn't a single literal field — pair `Discriminator(fn)` with `Tag(...)`:

```python
from typing import Annotated, Any, Literal, Optional, Union

from pydantic import BaseModel, Discriminator, Tag


class ApplePie(BaseModel):
    fruit: Literal['apple'] = 'apple'


class PumpkinPie(BaseModel):
    filling: Literal['pumpkin'] = 'pumpkin'


def get_discriminator_value(v: Any) -> Optional[str]:
    if isinstance(v, dict):
        return v.get('fruit', v.get('filling'))
    return getattr(v, 'fruit', getattr(v, 'filling', None))


class ThanksgivingDinner(BaseModel):
    dessert: Annotated[
        Union[
            Annotated[ApplePie, Tag('apple')],
            Annotated[PumpkinPie, Tag('pumpkin')],
        ],
        Discriminator(get_discriminator_value),
    ]
```

## Syntax cheat sheet

```python
# string discriminator
some_field: Union[...] = Field(discriminator='my_discriminator')
some_field: Annotated[Union[...], Field(discriminator='my_discriminator')]

# callable Discriminator
some_field: Union[...] = Field(discriminator=Discriminator(...))
some_field: Annotated[Union[...], Discriminator(...)]
```

## Custom errors and Tags

Plain unions explode into N errors per nesting level; discriminated unions emit a single targeted error. Customise it on `Discriminator`:

```python
Discriminator(
    model_x_discriminator,
    custom_error_type='invalid_union_member',
    custom_error_message='Invalid union member',
    custom_error_context={'discriminator': 'str_or_model'},
)
```

Even without a discriminator, `Tag(...)` gives readable error paths:

```python
tag_adapter = TypeAdapter(
    Union[
        Annotated[DoubledList, Tag('DoubledList')],
        Annotated[StringsMap, Tag('StringsMap')],
    ]
)
```

`X | Y` and `Union[X, Y]` are equivalent here. See [[Pydantic CO 02 - Fields]], [[Pydantic CO 14 - Type Adapter]], [[Pydantic ER 02 - Validation Errors]].

💡 **Takeaway:** Reach for `Field(discriminator='kind')` the moment a union holds two or more models — performance, errors, and JSON schema all improve at once.
