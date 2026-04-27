# CO 02 — Fields

🔑 **Key insight:** `Field()` carries everything that isn't the type — defaults, constraints, aliases, JSON Schema metadata, mutability — and the `Annotated[type, Field(...)]` pattern is the type-checker-friendly way to attach it.

## Two ways to attach `Field`

```python
from pydantic import BaseModel, Field

class Model(BaseModel):
    name: str = Field(frozen=True)
```

```python
from typing import Annotated
from pydantic import BaseModel, Field, WithJsonSchema

class Model(BaseModel):
    name: Annotated[str, Field(strict=True), WithJsonSchema({'extra': 'data'})]
```

⚠️ Static type checkers don't understand `default`, `default_factory`, or `alias` inside `Annotated[..., Field(...)]` — use the assignment form (`x: int = Field(default=...)`) for those.

## Inspecting fields

```python
from typing import Annotated
from pydantic import BaseModel, Field, WithJsonSchema

class Model(BaseModel):
    a: Annotated[
        int, Field(gt=1), WithJsonSchema({'extra': 'data'}), Field(alias='b')
    ] = 1

field_info = Model.model_fields['a']
print(field_info.annotation)
#> <class 'int'>
print(field_info.alias)
#> b
print(field_info.metadata)
#> [Gt(gt=1), WithJsonSchema(json_schema={'extra': 'data'}, mode=None)]
```

## Defaults

```python
class User(BaseModel):
    name: str = 'John Doe'
    age: int = Field(default=20)
```

### `default_factory`

```python
from uuid import uuid4
from pydantic import BaseModel, Field

class User(BaseModel):
    id: str = Field(default_factory=lambda: uuid4().hex)
```

The factory may take previously-validated data:

```python
from pydantic import BaseModel, EmailStr, Field

class User(BaseModel):
    email: EmailStr
    username: str = Field(default_factory=lambda data: data['email'])

user = User(email='[email protected]')
print(user.username)
#> [email protected]
```

### Validating defaults

```python
from pydantic import BaseModel, Field, ValidationError

class User(BaseModel):
    age: int = Field(default='twelve', validate_default=True)
# raises ValidationError on User()
```

Defaults are not validated by default. Set `validate_default=True` to opt in.

### Mutable defaults are deep-copied per instance

```python
class Model(BaseModel):
    item_counts: list[dict[str, int]] = [{}]

m1 = Model()
m1.item_counts[0]['a'] = 1
print(m1.item_counts)  #> [{'a': 1}]
m2 = Model()
print(m2.item_counts)  #> [{}]
```

No shared-state footgun — Pydantic deep-copies unhashable defaults.

## Aliases — see [[Pydantic CO 07 - Alias]]

```python
class User(BaseModel):
    name: str = Field(alias='username')          # both directions
class User(BaseModel):
    name: str = Field(validation_alias='username')   # input only
class User(BaseModel):
    name: str = Field(serialization_alias='username')  # output only
```

`validation_alias` beats `alias` for input; `serialization_alias` beats `alias` for output.

## Constraints

```python
from decimal import Decimal
from pydantic import BaseModel, Field

class Model(BaseModel):
    positive: int = Field(gt=0)
    short_str: str = Field(max_length=3)
    precise_decimal: Decimal = Field(max_digits=5, decimal_places=2)
```

Numeric: `gt`, `ge`, `lt`, `le`, `multiple_of`, `allow_inf_nan`. Strings: `min_length`, `max_length`, `pattern`. Decimal: `max_digits`, `decimal_places`. Collections: `min_length`, `max_length`.

### Constraints on unions

```python
from typing import Annotated, Union
class Model(BaseModel):
    positive: Union[int, None] = Field(gt=0)
    negative: Annotated[Union[int, None], Field(lt=0)]
```

⚠️ `Annotated[int, Field(gt=0)] | None` applies the constraint to the **type**, not the field — different semantics from `Field()` on the union itself.

## Strict per-field

```python
class User(BaseModel):
    name: str = Field(strict=True)
    age: int = Field(strict=False)

user = User(name='John', age='42')
#> name='John' age=42
```

## Dataclass-only knobs

```python
from pydantic import BaseModel, Field
from pydantic.dataclasses import dataclass

@dataclass
class Foo:
    bar: str
    baz: str = Field(init_var=True)
    qux: str = Field(kw_only=True)
```

`init_var` and `kw_only` come from stdlib dataclass semantics — see [[Pydantic CO 11 - Dataclasses]].

## `repr`

```python
class User(BaseModel):
    name: str = Field(repr=True)
    age: int = Field(repr=False)

print(User(name='John', age=42))
#> name='John'
```

## Discriminated unions

By field name:

```python
from typing import Literal, Union

class Cat(BaseModel):
    pet_type: Literal['cat']
    age: int

class Dog(BaseModel):
    pet_type: Literal['dog']
    age: int

class Model(BaseModel):
    pet: Union[Cat, Dog] = Field(discriminator='pet_type')
```

By callable — when variants don't share a tag name:

```python
from typing import Annotated, Literal, Union
from pydantic import BaseModel, Discriminator, Field, Tag

class Cat(BaseModel):
    pet_type: Literal['cat']; age: int

class Dog(BaseModel):
    pet_kind: Literal['dog']; age: int

def pet_discriminator(v):
    if isinstance(v, dict):
        return v.get('pet_type', v.get('pet_kind'))
    return getattr(v, 'pet_type', getattr(v, 'pet_kind', None))

class Model(BaseModel):
    pet: Union[Annotated[Cat, Tag('cat')], Annotated[Dog, Tag('dog')]] = Field(
        discriminator=Discriminator(pet_discriminator)
    )
```

See [[Pydantic CO 06 - Unions]].

## `frozen` per-field

```python
class User(BaseModel):
    name: str = Field(frozen=True)
    age: int
# user.name = 'Jane' -> ValidationError, type=frozen_field
```

## Excluding from output

```python
class User(BaseModel):
    name: str
    age: int = Field(exclude=True)

print(User(name='John', age=42).model_dump())
#> {'name': 'John'}
```

## Deprecated

```python
from typing import Annotated
class Model(BaseModel):
    deprecated_field: Annotated[int, Field(deprecated='This is deprecated')]
```

Or with `typing_extensions.deprecated`:

```python
from typing_extensions import deprecated
class Model(BaseModel):
    deprecated_field: Annotated[int, deprecated('This is deprecated')]
    alt_form: Annotated[int, Field(deprecated=deprecated('This is deprecated'))]
```

⚠️ Pydantic ignores `category` and `stacklevel` on `deprecated`. Reading the field emits `DeprecationWarning` — silence inside validators with `warnings.catch_warnings()`.

## Computed fields

```python
from pydantic import BaseModel, computed_field

class Box(BaseModel):
    width: float
    height: float
    depth: float

    @computed_field
    @property
    def volume(self) -> float:
        return self.width * self.height * self.depth

b = Box(width=1, height=2, depth=3)
print(b.model_dump())
#> {'width': 1.0, 'height': 2.0, 'depth': 3.0, 'volume': 6.0}
```

Appears in serialization output and serialization-mode JSON Schema. ⚠️ No automatic cache invalidation — Pydantic doesn't wrap your `@property`.

💡 **Takeaway:** Reach for `Field()` whenever the type alone can't carry the constraint, default, or schema hint — `Annotated` keeps it composable.
