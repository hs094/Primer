# 13 — Extra Models & Status Codes

## Union / discriminated response

```python
from typing import Union  # or X | Y

class Car(BaseModel):
    type: Literal["car"]
    wheels: int

class Plane(BaseModel):
    type: Literal["plane"]
    engines: int

@app.get("/vehicle", response_model=Union[Car, Plane])
async def get(): ...
```

- Put more-specific models first in the Union, or use a `Field(discriminator="type")` for deterministic tagged unions.

## List / dict response model

```python
@app.get("/items/", response_model=list[Item])
@app.get("/keyword-weights/", response_model=dict[str, float])
```

## Status code

```python
from fastapi import status

@app.post("/items/", status_code=status.HTTP_201_CREATED)
async def create(...): ...
```

- Use `fastapi.status` (or `starlette.status`) constants, not magic numbers.
- Default is `200 OK`.
- For `204`, return `None` or an empty body; FastAPI sends no body.

## Custom per-route status without `HTTPException`

See [[Advanced/03 - Additional Responses]] for `response.status_code = ...` inside the handler.

## Gotcha

- ⚠️ `Union` response models can produce ambiguous validation errors — prefer a `discriminator=` to route explicit types.
