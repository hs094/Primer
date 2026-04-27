# CO 12 — Forward Annotations

🔑 **Key insight:** Pydantic resolves string / `from __future__ import annotations` types at class-creation time — but when a referenced name isn't in scope yet, you call `model_rebuild()` to finish the job.

## Two ways to forward-reference

```python
from __future__ import annotations

from pydantic import BaseModel

MyInt = int

class Model(BaseModel):
    a: MyInt

print(Model(a='1'))
#> a=1
```

Or per-annotation string quoting — equivalent for Pydantic's purposes:

```python
from typing import Optional

from pydantic import BaseModel

class Foo(BaseModel):
    a: int = 123
    sibling: 'Optional[Foo]' = None

print(Foo())
#> a=123 sibling=None
print(Foo(sibling={'a': '321'}))
#> a=123 sibling=Foo(a=321, sibling=None)
```

Self-referencing fields are supported; annotations resolve during model creation.

## Why string annotations need rebuilding

Annotations are stored as strings (PEP 563 / quoted). Pydantic must `eval` them against a namespace to get the real types. If a referenced name isn't yet defined when the class body runs — common for mutually-recursive models or imports inside `TYPE_CHECKING` — Pydantic defers and you must call `model_rebuild()` once the names exist.

The full resolution algorithm lives in [[Pydantic IT 02 - Resolving Annotations]].

## Cyclic references → `ValidationError`, not `RecursionError`

```python
from typing import Optional

from pydantic import BaseModel, ValidationError

class ModelA(BaseModel):
    b: 'Optional[ModelB]' = None

class ModelB(BaseModel):
    a: Optional[ModelA] = None

cyclic_data = {}
cyclic_data['a'] = {'b': cyclic_data}

try:
    ModelB.model_validate(cyclic_data)
except ValidationError as exc:
    print(exc)
```

The error type is `'recursion_loop'`. See [[Pydantic ER 01 - Error Handling]] for catching it.

## Mutually-recursive models pattern

```python
from __future__ import annotations
from pydantic import BaseModel

class A(BaseModel):
    b: B | None = None

class B(BaseModel):
    a: A | None = None

# A's annotation 'B' was unresolved at class-creation; rebuild now that B exists.
A.model_rebuild()
```

If `model_rebuild()` is omitted, the first validation attempt that needs `B` will raise a `PydanticUndefinedAnnotation`-style error.

## Custom namespace for `model_rebuild`

When the type lives somewhere `eval` can't find via the class's module globals (factory functions, dynamic types, `TYPE_CHECKING` imports), pass an explicit namespace:

```python
A.model_rebuild(_types_namespace={'B': B})
```

This is the escape hatch when normal scope-walking fails.

## `defer_build` — opt out of build-at-class-creation

```python
from pydantic import BaseModel, ConfigDict

class Heavy(BaseModel):
    model_config = ConfigDict(defer_build=True)
    # ... lots of fields
```

The schema is constructed on first use (or on explicit `model_rebuild()`). Useful for circular-import scenarios and startup-time-sensitive apps. See [[Pydantic CO 18 - Performance]].

The same flag exists on [[Pydantic CO 14 - Type Adapter]] — schema is built lazily and you call `.rebuild()`.

## Suppressing recursion errors inside validators / serializers

For genuinely cyclic graphs you want to walk yourself, the docs include patterns for catching `'recursion_loop'` inside field validators and using field serializers to break the cycle on dump. The general shape: catch the cycle marker, return a sentinel (`None`, `'<cycle>'`, an id reference) instead of recursing.

## When you do *not* need rebuild

- Forward refs that resolve to names already in module globals at class-creation.
- Self-references where `from __future__ import annotations` is set and the class name is the only forward ref — Pydantic handles the self-ref automatically.
- Generic models parameterized only with concrete builtins.

⚠️ **Footgun:** importing a model whose forward refs depend on a sibling module *not yet imported* will silently leave the schema in a partially-built state. The error surfaces only when you try to validate. If you have a circular import graph, call `model_rebuild()` in your `__init__.py` after both modules are loaded.

## Quick reference

```python
# After defining mutually-recursive models:
A.model_rebuild()

# When the referenced type isn't reachable from A's module globals:
A.model_rebuild(_types_namespace={'B': B, 'Helper': Helper})

# For TypeAdapter:
ta = TypeAdapter('MyType', config=ConfigDict(defer_build=True))
# ... later, after MyType exists ...
ta.rebuild()
```

💡 **Takeaway:** Forward refs are fine until they aren't — when they aren't, `model_rebuild()` (optionally with `_types_namespace`) is the one tool you need.
