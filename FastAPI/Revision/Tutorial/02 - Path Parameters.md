# 02 — Path Parameters

🔑 Declared in the URL template and as a function argument of the same name. Type hints trigger validation.

## Basic

```python
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}
```

- `int`, `float`, `bool`, `str`, `UUID`, `date`, etc. are validated + coerced.
- Mismatched type → `422 Unprocessable Entity` with a structured error.

## Order matters

Routes are matched top-to-bottom. Put **fixed** paths before **dynamic** ones.

```python
@app.get("/users/me")     # MUST be above
@app.get("/users/{id}")
```

## `Enum` for a closed set

```python
from enum import Enum

class Model(str, Enum):
    alexnet = "alexnet"
    resnet  = "resnet"

@app.get("/models/{name}")
async def get_model(name: Model):
    if name is Model.alexnet: ...
```

- Subclass `str, Enum` so it serializes as a string.
- Validation rejects values not in the enum.

## `:path` converter — capture slashes

```python
@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    # /files/home/me/x.txt → file_path = "home/me/x.txt"
```

- For absolute paths start URL with `/files//home/...` (Starlette quirk).

## vs Path validation

See [[05 - Query and Path Validation]] for `Path(..., ge=1, le=1000, title=..., description=...)` — adds metadata + numeric constraints without changing function signature semantics.

## Gotchas

- ⚠️ Untyped path params are treated as `str`.
- ⚠️ Python bool from path accepts `1/0, true/false, yes/no, on/off` (case-insensitive).
- 💡 If you use `Annotated[int, Path(ge=1)]`, prefer putting `Path`/`Query` in `Annotated` rather than as default values.
