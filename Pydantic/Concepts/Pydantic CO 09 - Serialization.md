# CO 09 — Serialization

🔑 **Key insight:** `model_dump()` returns Python objects; `model_dump(mode='json')` returns JSON-safe Python; `model_dump_json()` returns the actual string — pick the one that matches what consumes the output.

## The three core methods

```python
from typing import Optional

from pydantic import BaseModel, Field


class BarModel(BaseModel):
    whatever: tuple[int, ...]


class FooBarModel(BaseModel):
    banana: Optional[float] = 1.1
    foo: str = Field(serialization_alias='foo_alias')
    bar: BarModel


m = FooBarModel(banana=3.14, foo='hello', bar={'whatever': (1, 2)})

print(m.model_dump())
#> {'banana': 3.14, 'foo': 'hello', 'bar': {'whatever': (1, 2)}}

print(m.model_dump(by_alias=True))
#> {'banana': 3.14, 'foo_alias': 'hello', 'bar': {'whatever': (1, 2)}}

print(m.model_dump(mode='json'))
#> {'banana': 3.14, 'foo': 'hello', 'bar': {'whatever': [1, 2]}}
```

`mode='json'` coerces tuples → lists, datetimes → ISO strings, etc., but still returns a `dict`. `model_dump_json(indent=2)` returns a JSON string directly. See [[Pydantic CO 07 - Alias]], [[Pydantic CO 16 - Conversion Table]].

⚠️ `dict(model)` and iteration do **not** recurse — sub-models stay as model instances. Use `model_dump()` if you need a fully-converted dict.

## Field serializers

Two modes: **plain** (replaces Pydantic's logic) and **wrap** (delegates via `handler`). One serializer per field max.

Plain, annotated:

```python
from typing import Annotated, Any

from pydantic import BaseModel, PlainSerializer


def ser_number(value: Any) -> Any:
    if isinstance(value, int):
        return value * 2
    else:
        return value


class Model(BaseModel):
    number: Annotated[int, PlainSerializer(ser_number)]


print(Model(number=4).model_dump())
#> {'number': 8}
```

Wrap, annotated:

```python
from typing import Annotated, Any

from pydantic import BaseModel, SerializerFunctionWrapHandler, WrapSerializer


def ser_number(value: Any, handler: SerializerFunctionWrapHandler) -> int:
    return handler(value) + 1


class Model(BaseModel):
    number: Annotated[int, WrapSerializer(ser_number)]


print(Model(number=4).model_dump())
#> {'number': 5}
```

Decorator form — same modes, but bind to one or more fields on a model:

```python
class Model(BaseModel):
    f1: str
    f2: str

    @field_serializer('f1', 'f2', mode='plain')
    def capitalize(self, value: str) -> str:
        return value.capitalize()
```

## Model serializers

`@model_serializer` reshapes the whole output. Plain replaces; wrap augments via `handler`.

```python
from pydantic import BaseModel, SerializerFunctionWrapHandler, model_serializer


class UserModel(BaseModel):
    username: str
    password: str

    @model_serializer(mode='wrap')
    def serialize_model(
        self, handler: SerializerFunctionWrapHandler
    ) -> dict[str, object]:
        serialized = handler(self)
        serialized['fields'] = list(serialized)
        return serialized


print(UserModel(username='foo', password='bar').model_dump())
#> {'username': 'foo', 'password': 'bar', 'fields': ['username', 'password']}
```

## Serialization context

Pass a `context` to `model_dump`; access via `info.context` inside any serializer:

```python
from pydantic import BaseModel, FieldSerializationInfo, field_serializer


class Model(BaseModel):
    text: str

    @field_serializer('text', mode='plain')
    @classmethod
    def remove_stopwords(cls, v: str, info: FieldSerializationInfo) -> str:
        if isinstance(info.context, dict):
            stopwords = info.context.get('stopwords', set())
            v = ' '.join(w for w in v.split() if w.lower() not in stopwords)
        return v


print(Model(text='This is an example').model_dump(
    context={'stopwords': ['this', 'is', 'an']}
))
#> {'text': 'example'}
```

`info` also exposes the current `mode` (`'python'` / `'json'`) and field name.

## Subclass behaviour

⚠️ When a field is annotated `User` but holds a `UserLogin`, **only `User`'s declared fields serialize** — subclass fields are dropped (info-leak prevention, footgun if unintended):

```python
class User(BaseModel):
    name: str


class UserLogin(User):
    password: str


class OuterModel(BaseModel):
    user: User


m = OuterModel(user=UserLogin(name='pydantic', password='hunter2'))
print(m.model_dump())
#> {'user': {'name': 'pydantic'}}
```

Opt out per-field with `SerializeAsAny[User]`, per-call with `m.model_dump(serialize_as_any=True)`, or `m.model_dump(polymorphic_serialization=True)` (v2.13+).

## Include / exclude

### At the field

```python
from pydantic import BaseModel, Field


class Transaction(BaseModel):
    id: int
    private_id: int = Field(exclude=True)
    value: int = Field(ge=0, exclude_if=lambda v: v == 0)


print(Transaction(id=1, private_id=2, value=0).model_dump())
#> {'id': 1}
```

### At the call site

```python
print(t.model_dump(exclude={'user', 'value'}))
#> {'id': '1234567890'}

print(t.model_dump(exclude={'user': {'username', 'password'}, 'value': True}))
#> {'id': '1234567890', 'user': {'id': 42}}

print(t.model_dump(include={'id': True, 'user': {'id'}}))
#> {'id': '1234567890', 'user': {'id': 42}}
```

For collections, use index keys or `'__all__'`:

```python
print(user.model_dump(exclude={'hobbies': {'__all__': {'info'}}}))
#> {'hobbies': [{'name': 'Programming'}, {'name': 'Gaming'}]}
```

### Value-based filters

- `exclude_defaults` — drop fields equal to default.
- `exclude_none` — drop fields whose value is `None`.
- `exclude_unset` — drop fields not explicitly set on construction.

```python
class UserModel(BaseModel):
    name: str
    age: int = 18


user = UserModel(name='John')
print(user.model_dump(exclude_unset=True))
#> {'name': 'John'}

user.age = 21
print(user.model_dump(exclude_unset=True))
#> {'name': 'John', 'age': 21}
```

⚠️ Assigning to a field marks it as set — `exclude_unset` after a write returns it.

## Computed fields

`@computed_field` exposes derived properties as serialized fields. See [[Pydantic CO 02 - Fields]] for the full signature; in short, decorate a `@property` so it shows up in `model_dump()` and JSON schema. Pair with [[Pydantic CO 03 - JSON Schema]] when the output schema must reflect the derived value.

See [[Pydantic CO 10 - Validators]] for the read-side analogue, [[Pydantic CO 16 - Conversion Table]] for what `mode='json'` actually does to each type, and [[Pydantic CO 18 - Performance]] for dump-perf tips.

💡 **Takeaway:** Default to `model_dump_json()` at API boundaries, `model_dump()` for internal handoff, and reach for `WrapSerializer` only when you need to *augment* rather than replace Pydantic's output.
