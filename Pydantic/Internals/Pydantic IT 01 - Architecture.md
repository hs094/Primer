# IT 01 — Architecture

🔑 **Key insight:** Pydantic V2 is a thin Python frontend over a Rust core (`pydantic-core`); they communicate exclusively through a single serializable Python dict — the **core schema** — which is why everything from validation to JSON-schema export hangs off three model attributes.

The two-package split is what unlocks the *"5 to 20 performance increase compared to Pydantic V1"* the docs cite (see [[Pydantic CO 18 - Performance]]).

## The pieces

- **`pydantic`** (Python) — model definition, metaclass machinery, decorators, schema generation, JSON-schema generation.
- **`pydantic-core`** (Rust) — runtime validation and serialization, exposed to Python as `SchemaValidator` and `SchemaSerializer`.

## What happens when you define a model

When you subclass [[Pydantic CO 01 - Models|`BaseModel`]], the metaclass collects:

- field annotations (later available as `model_fields`)
- model configuration via `model_config` (see [[Pydantic CO 08 - Configuration]])
- custom validators and serializers ([[Pydantic CO 10 - Validators]])
- private attributes and generic parameters

It then asks `GenerateSchema` to walk those annotations and emit a **core schema**.

## The core schema

A core schema is *"a structured (and serializable) Python dictionary"* expressed via `TypedDict` definitions. Every node has a `type` key plus type-specific properties.

For a simple model:

```python
from pydantic import BaseModel, Field

class Model(BaseModel):
    foo: bool = Field(strict=True)
```

…the schema fragment for `foo` looks like:

```python
{
    'type': 'bool',
    'strict': True,
}
```

Custom serializers feed in as their own schema nodes. Given:

```python
class Model(BaseModel):
    foo: bool = Field(strict=True)

    @field_serializer('foo', mode='plain')
    def serialize_foo(self, value: bool) -> Any:
        ...
```

…the serialization side emits:

```python
{
    'type': 'function-plain',
    'function': <function Model.serialize_foo at 0x111>,
    'is_field_serializer': True,
    'info_arg': False,
    'return_schema': {'type': 'int'},
}
```

## The three model attributes

Every model class ends up carrying:

| Attribute | Produced by | Consumed by |
|---|---|---|
| `__pydantic_core_schema__` | `GenerateSchema` | both validator and serializer (Rust) |
| `__pydantic_validator__` | `pydantic-core` (`SchemaValidator`) | `Model(...)`, `model_validate*` |
| `__pydantic_serializer__` | `pydantic-core` (`SchemaSerializer`) | `model_dump`, `model_dump_json` |

These are the public hooks if you ever need to introspect or override what the metaclass produced.

## JSON Schema is a separate transform

`GenerateJsonSchema` takes the core schema and emits standard JSON Schema. For the bool example above:

```python
{"type": "boolean"}
```

This is the same machinery [[Pydantic IN 02 - LLMs|`Model.model_json_schema()`]] uses — JSON Schema is *derived* from core schema, never the other way around.

## Customising schema generation

Two paired methods let you intercept generation for annotated types:

- `__get_pydantic_core_schema__` — override what core schema is built
- `__get_pydantic_json_schema__` — override the JSON-schema projection

Wrapper pattern (note `handler(source)` to delegate, then mutate):

```python
from typing import Annotated, Any
from pydantic_core import CoreSchema
from pydantic import GetCoreSchemaHandler, TypeAdapter

class MyStrict:
    @classmethod
    def __get_pydantic_core_schema__(
        cls, source: Any, handler: GetCoreSchemaHandler
    ) -> CoreSchema:
        schema = handler(source)
        schema['strict'] = True
        return schema

ta = TypeAdapter(Annotated[int, MyStrict()])
```

This is the idiomatic way to tag types with extra constraints — Pydantic itself uses it for [[Pydantic CO 05 - Types|its built-in types]].

## Runtime: validation and serialization

Once a model is fully built, all runtime work goes through the Rust layer:

- `SchemaValidator.validate_python` — validates input data using the core schema
- `SchemaSerializer.to_python` — serializes a model instance using the core schema

You almost never call these directly; the model methods are wrappers:

```python
class Model(BaseModel):
    foo: int

model = Model.model_validate({'foo': 1})  # Uses SchemaValidator
dumped = model.model_dump()                # Uses SchemaSerializer
```

The validator populates the instance's `__dict__`, and the serializer reads from `__dict__` to produce output — meaning anything you cache or compute in `model_post_init` is visible to serialization.

## Why this matters

- The split is the reason [[Pydantic CO 14 - Type Adapter|`TypeAdapter`]] exists: any type with a core schema can be validated, no `BaseModel` required.
- It's also why [[Pydantic IT 02 - Resolving Annotations|forward-ref resolution]] is its own concern — schema generation is the gate that turns Python annotations into a Rust-consumable structure.
- And it's why custom types should hook `__get_pydantic_core_schema__` rather than reimplement validation in `__init__`.

⚠️ The core schema is an internal contract — keys can shift between minor versions. Lean on the public `GetCoreSchemaHandler` API rather than hand-building dicts.

💡 **Takeaway:** Annotation → `GenerateSchema` → core-schema dict → Rust `SchemaValidator`/`SchemaSerializer`. Everything else (JSON Schema, model validation, serialization) is a function of that one dict.
