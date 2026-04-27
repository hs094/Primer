# CO 15 — Validation Decorator

🔑 **Key insight:** `@validate_call` parses and validates a function's arguments against its type hints on every call — same engine as `BaseModel`, applied at the function boundary.

## Basic usage

```python
from pydantic import ValidationError, validate_call

@validate_call
def repeat(s: str, count: int, *, separator: bytes = b'') -> bytes:
    b = s.encode()
    return separator.join(b for _ in range(count))
```

Coercion fires by default — pass `'2000-01-01'` to a `date`-annotated parameter and you get a `date` object inside the function body.

## Failure mode: `ValidationError`, not `TypeError`

⚠️ Validation failures (including missing required arguments) raise `ValidationError`, **not** `TypeError`. If your callers were catching `TypeError` for missing-arg scenarios, that handler will no longer fire. See [[Pydantic ER 01 - Error Handling]].

## Supported parameter kinds

The decorator handles every parameter shape:

- Positional-or-keyword, with/without defaults
- Keyword-only (after `*`)
- Positional-only (before `/`)
- Variable positional (`*args`) and variable keyword (`**kwargs`)
- `Unpack[SomeTypedDict]` for type-safe `**kwargs`

## Field constraints on parameters

Use `Annotated` + `Field()` to apply per-field constraints — same as in [[Pydantic CO 02 - Fields]]:

```python
from typing import Annotated
from pydantic import Field, validate_call

@validate_call
def how_many(num: Annotated[int, Field(gt=10)]):
    return num
```

## Validating return values

By default the return value is **not** validated (function trusts itself). Opt in:

```python
@validate_call(validate_return=True)
def make_user(name: str) -> User:
    ...
```

When the function's return annotation is wrong or its body returns the wrong shape, you'll see a `ValidationError` at call time instead of a downstream bug.

## Configuration

Pass a `ConfigDict` for the same knobs as a model — see [[Pydantic CO 08 - Configuration]]:

```python
@validate_call(config=ConfigDict(arbitrary_types_allowed=True))
def add_foobars(a: Foobar, b: Foobar):
    return a + b
```

Strict mode works too — annotate parameters with `Strict()` / `StrictInt` / etc. or set `strict=True` in the config. See [[Pydantic CO 13 - Strict Mode]].

## Async functions

```python
@validate_call
async def fetch(url: str, timeout: float = 5.0) -> bytes:
    ...
```

Works with no special handling — the wrapper just `await`s.

## Escape hatch: `raw_function`

When you need the un-validated original (tests, perf-critical inner loops):

```python
result = decorated_func.raw_function(arg1, arg2)
```

## When the overhead is worth it

**Yes:**
- Public package APIs where bad inputs from end-users should fail loudly with rich errors instead of cryptic `TypeError`s deep in the call stack.
- Top-level CLI entry points (combine with [[Pydantic CO 14 - Type Adapter]] for argv parsing).
- Boundary functions that take JSON-shaped dicts and would otherwise need manual `BaseModel(**kwargs)` construction.

**No:**
- Hot inner-loop functions called millions of times per request — overhead per call is real and additive. Validate at the boundary instead.
- Functions whose inputs are already validated upstream (e.g. handler bodies after FastAPI has already run Pydantic). Double-validating is wasted CPU.
- Functions returning complex types where `validate_return=True` would re-walk the entire output tree on every call.

The docs are explicit: `@validate_call` "is not an equivalent or alternative to function definitions in strongly typed languages."

## Composition with constraints

```python
from typing import Annotated
from pydantic import Field, ConfigDict, validate_call

@validate_call(config=ConfigDict(strict=True), validate_return=True)
def percentile(
    values: list[float],
    p: Annotated[float, Field(ge=0.0, le=1.0)],
) -> float:
    s = sorted(values)
    return s[int(p * (len(s) - 1))]
```

One decorator gives you: type coercion off (strict), constraint check on `p`, list element types validated, return value validated.

## What you don't get

- No automatic `model_dump`-style serialization of arguments — this is a function decorator, not a model.
- No `__pydantic_fields__` on the wrapper. If you need introspection, use `BaseModel`.
- No partial validation / "fill missing from defaults via env" behavior — that's `pydantic-settings` territory.

💡 **Takeaway:** Use `@validate_call` at the *outer* boundary of a module — public APIs, CLIs, message handlers — where the input is untrusted; skip it on the inner-loop helpers it calls.
