# IT 02 — Resolving Annotations

🔑 **Key insight:** Pydantic resolves forward-reference annotations by walking the class's MRO and a fixed namespace stack at schema-generation time; if any name is still missing the model becomes a `MockCoreSchema` placeholder until you call `model_rebuild()`.

This is the runtime cost of `from __future__ import annotations` — see [[Pydantic CO 12 - Forward Annotations]] for the user-facing knobs and [[Pydantic IT 01 - Architecture|the architecture page]] for where schema generation sits in the pipeline.

## Why it matters

Python stringifies type hints under PEP 563 / `from __future__ import annotations`. The docs put it bluntly: *"For runtime tools such as Pydantic, it is more challenging to correctly resolve these forward annotations."* Static checkers see them as text; Pydantic has to `eval()` them into real types.

## The namespace stack

When a model is built, annotations are resolved by looking up names in this order (highest priority first):

1. A namespace containing the current class itself (`{cls.__name__: cls}`) — supports recursive references
2. The class's own `__dict__` — lets `LocalType` defined inside the class be found
3. The parent namespace (frame locals where the class was defined, if different from globals)
4. The module globals (`sys.modules[module_name].__dict__`)

The class's MRO contributes its own bases' namespaces too, so a forward ref defined in a parent module is reachable from a subclass declared in another module.

## Worked example

```python
# module1.py
type MyType = int

class Base:
    f1: 'MyType'

# module2.py
from pydantic import BaseModel
from module1 import Base

type MyType = str

def inner() -> None:
    type InnerType = bool

    class Model(BaseModel, Base):
        type LocalType = bytes

        f2: 'MyType'      # Resolves to str
        f3: 'InnerType'   # Resolves to bool
        f4: 'LocalType'   # Resolves to bytes
        f5: 'UnknownType' # Remains unresolved
```

Resolution table:

| Field | Resolves to | Source |
|---|---|---|
| `f1` | `int` | module1's `MyType` (carried via `Base`) |
| `f2` | `str` | module2's `MyType` (caller globals) |
| `f3` | `bool` | `inner()` frame locals |
| `f4` | `bytes` | `Model`'s own `__dict__` |
| `f5` | unresolved | not in any layer of the stack |

`f5` doesn't raise — Pydantic stashes it as a `MockCoreSchema`.

## When resolution fails: `MockCoreSchema`

If any annotation fails to resolve, the whole model schema becomes a placeholder:

```python
class Foo(BaseModel):
    f: 'MyType'

print(Foo.__pydantic_core_schema__)
#> <pydantic._internal._mock_val_ser.MockCoreSchema object at ...>
```

Calling `Foo(...)` against a `MockCoreSchema` raises [[Pydantic ER 03 - Usage Errors|`PydanticUserError` (`class-not-fully-defined`)]] — the model isn't usable yet.

## Repairing with `model_rebuild()`

Once the missing name exists, force re-resolution:

```python
type MyType = int

Foo.model_rebuild()
# Now f: 'MyType' successfully resolves to int
```

Rebuild namespace semantics:

- If `_types_namespace` is explicitly passed, that dict *is* the rebuild namespace.
- Otherwise, the *caller's* namespace is used (frame locals + globals at the call site).
- Either way, this namespace is merged with the model's parent namespace if it was defined inside a function.

Explicit namespace example:

```python
class Foo(BaseModel):
    f: 'MyType'

Foo.model_rebuild(_types_namespace={'MyType': int})
```

## Limitations called out by the docs

> "When the Model class is being created inside a function, we keep a copy of the locals of the frame. This copy only includes symbols defined when Model is being defined."

So a name bound *after* the class statement inside the same function is invisible to schema generation; you must `Model.model_rebuild()` once it's bound.

Other gotchas:

- A class's `__dict__` may include dunder attributes — `f: '__doc__'` will unexpectedly resolve to the docstring.
- ⚠️ Forward references may not resolve outside the original function because Pydantic intentionally uses weak references to avoid memory leaks — bind any out-of-scope types via `_types_namespace`.
- ⚠️ Dataclasses, `TypedDict`s, and `NamedTuple`s do **not** apply the same locals-handling pattern — their annotations need to be resolvable from globals or you must hand them a namespace.

## Backwards-compatibility hack

For interop, Pydantic injects `{Bar.__name__: Bar}` into locals while building schemas for *other* classes — even while `Bar` is itself still being created. This means common self-referential and mutual-reference patterns Just Work without manual `model_rebuild()`, but it can introduce subtle inconsistencies if you depend on a strict layered build order.

## `defer_build` for control

If you'd rather *not* eagerly build the schema (e.g. during library import where types may not all be in scope yet), set `model_config = ConfigDict(defer_build=True)` (see [[Pydantic CO 08 - Configuration]]). The schema is then built on first use, by which point all forward refs are typically resolvable.

⚠️ With `defer_build=True`, the *first* validation pays the schema-build cost — call `model_rebuild()` once at startup if you need predictable latency.

💡 **Takeaway:** Forward-ref resolution walks `cls itself → cls.__dict__ → parent frame locals → module globals`; if any annotation slips through the net you get a `MockCoreSchema` and `model_rebuild()` (optionally with `_types_namespace`) is the fix.
