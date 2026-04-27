# Adv 14 — Additional Status Codes

🔑 The default route status code is fixed by `status_code=…` on the decorator, but you can return a *different* status from inside the handler by emitting a `JSONResponse` directly.

## Return a different status mid-handler

```python
from typing import Annotated
from fastapi import FastAPI, Body, status
from fastapi.responses import JSONResponse

app = FastAPI()
items = {"foo": {"name": "Foo"}}

@app.put("/items/{item_id}")
async def upsert(item_id: str, name: Annotated[str, Body()]):
    if item_id in items:
        items[item_id]["name"] = name
        return items[item_id]                       # → 200 (the default)
    items[item_id] = {"name": name}
    return JSONResponse(content=items[item_id],
                        status_code=status.HTTP_201_CREATED)
```

- Default route status: 200.
- Created branch returns 201 explicitly.

## Why not `HTTPException`?

`HTTPException` is for *errors*. For successful responses with non-default status, return a `Response` / `JSONResponse`.

## OpenAPI implications

⚠️ FastAPI does **not** auto-detect the alternate status code in the schema — declare it via `responses=` ([[FastAPI 03 - Additional Responses]]):

```python
@app.put("/items/{id}", responses={201: {"model": Item}})
```

## Status code constants

```python
from fastapi import status
status.HTTP_201_CREATED
status.HTTP_204_NO_CONTENT
status.HTTP_409_CONFLICT
```

💡 Use `status.HTTP_*` constants — readable + lint-friendly.

## Returning 204

```python
from fastapi import Response

@app.delete("/items/{id}", status_code=204)
async def delete(id: str):
    items.pop(id, None)
    return Response(status_code=204)        # no body
```

⚠️ 204 must have **no body** — let the response be empty.
