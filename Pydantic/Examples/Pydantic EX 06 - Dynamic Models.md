# EX 06 — Dynamic Models

🔑 **Key insight:** `create_model()` is the runtime constructor for `BaseModel` subclasses — combine it with `model_fields` and `FieldInfo.asdict()` to mechanically derive new schemas (all-optional, alias-stripped, default-less) from an existing one.

## When a static class is not enough

Most models are written by hand. But sometimes the schema itself is computed:

- A "patch" version of an existing model where every field is optional.
- A model whose fields are read from config or a registry at startup.
- A request schema generated from a database table introspection.

`create_model()` is the official factory for these cases. See [[Pydantic CO 01 - Models]] for the static counterpart.

## Deriving an all-optional model

The canonical example: take any `BaseModel` and produce a sibling whose fields all default to `None`. Useful for `PATCH` endpoints where every field is optional but the constraints (`gt=1`, regexes, etc.) still apply when a value *is* supplied.

```python
from typing import Annotated

from pydantic import BaseModel, Field, create_model


def make_fields_optional(model_cls: type[BaseModel]) -> type[BaseModel]:
  new_fields = {}

  for f_name, f_info in model_cls.model_fields.items():
      f_dct = f_info.asdict()
      new_fields[f_name] = (
          Annotated[f_dct['annotation'] | None, *f_dct['metadata'], Field(**f_dct['attributes'])],
          None,
      )

  return create_model(
      f'{model_cls.__name__}Optional',
      __base__=model_cls,
      **new_fields,
  )
```

Three things are happening per field:

1. **`f_info.asdict()`** decomposes a `FieldInfo` into its three meaningful pieces — `annotation` (the type), `metadata` (validators, constraints attached via `Annotated`), and `attributes` (everything else: `title`, `description`, `examples`, …). See [[Pydantic CO 02 - Fields]] for the field anatomy.
2. **`Annotated[T | None, *metadata, Field(**attributes)]`** rebuilds the type with the original metadata preserved and `None` widened in.
3. **The `(annotation, default)` tuple** is `create_model`'s field-spec format. `None` becomes the default, making the field optional.

## Using it

```python
from pydantic import BaseModel, Field


class Model(BaseModel):
    a: Annotated[int, Field(gt=1)]


ModelOptional = make_fields_optional(Model)

m = ModelOptional()
print(m.a)
#> None
```

`ModelOptional` keeps the `gt=1` constraint — so `ModelOptional(a=0)` still fails — but `ModelOptional()` is now valid because `a` defaults to `None`.

## Why `__base__=model_cls`

Passing the original model as `__base__` makes the derived model **inherit validators and computed fields** from the parent. If you used `__base__=BaseModel` (the default), `@field_validator` / `@model_validator` / `@computed_field` on the original would not carry over. The redefined fields override the parent fields, but the surrounding behavior is preserved.

## Limitations

⚠️ **Static type checkers cannot see the optionality.** `mypy` / `pyright` only know the parent's annotations — they have no way to infer that `ModelOptional`'s fields are now `T | None`. If you need static guarantees, write the patched class by hand.

⚠️ **Do not copy `FieldInfo` instances and mutate them directly.** The docs explicitly warn: "this may work in some cases, it is **not** a supported pattern, and could break." `asdict()` + `Field(**attributes)` is the supported decomposition.

## Generalizing the pattern

The same skeleton — iterate `model_fields`, decompose with `asdict()`, recompose with `Annotated` and `Field(...)` — handles other transformations:

| Goal | Tweak |
|---|---|
| Strip defaults | omit the second tuple element, pass `...` (Ellipsis) |
| Strip aliases | remove `'alias'` from `f_dct['attributes']` before splatting |
| Add a prefix to all field names | mutate `f_name` before assigning into `new_fields` |
| Pick a subset | filter the loop |

For the runtime end of the pipeline (validating arbitrary payloads against a constructed model), pair this with [[Pydantic CO 14 - Type Adapter]]. Failures look like normal validation errors — see [[Pydantic ER 02 - Validation Errors]].

## Versions

The example uses `f_dct['annotation'] | None`, which is Python 3.10+ union syntax. On 3.9, swap for `Optional[f_dct['annotation']]` — but per global preference, 3.10+ syntax is canonical.

💡 **Takeaway:** `create_model` + `model_fields` + `FieldInfo.asdict()` is the supported runtime path for deriving schemas — keep `__base__` set to the parent so validators ride along, and accept that static type checkers will not follow you.
