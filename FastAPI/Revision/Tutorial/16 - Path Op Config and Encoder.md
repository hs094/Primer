# 16 — Path Op Config, JSON Encoder, PATCH Updates

## Path operation decorator kwargs

| Kwarg | Purpose |
|---|---|
| `status_code=201` | default response status |
| `tags=["items"]` | group in docs |
| `summary="..."` | short title in docs |
| `description="..."` | markdown; also pulled from docstring |
| `response_description="..."` | describes the response section |
| `deprecated=True` | mark deprecated in UI |
| `include_in_schema=False` | hide from OpenAPI |
| `operation_id="..."` | stable ID for codegen |
| `responses={404: {...}}` | extra responses (see [[Advanced/03 - Additional Responses]]) |

```python
@app.post("/items/", response_model=Item, status_code=201,
          tags=["items"], summary="Create an item",
          response_description="The created item")
async def create(item: Item):
    """Create an item.\n\n- markdown supported"""
    ...
```

## `jsonable_encoder`

Use when you must produce a JSON-ready primitive structure yourself (e.g., before shoving into a Mongo/Redis client).

```python
from fastapi.encoders import jsonable_encoder

data = jsonable_encoder(item)   # nested dict/list of JSON primitives
```

- Handles Pydantic models, `datetime`, `UUID`, `Enum`, `Decimal`, `bytes` (→ string), etc.

## PUT vs PATCH (partial updates)

```python
@app.patch("/items/{id}", response_model=Item)
async def patch(id: int, patch: ItemPatch):
    stored: Item = store[id]
    update_data = patch.model_dump(exclude_unset=True)
    updated = stored.model_copy(update=update_data)
    store[id] = updated
    return updated
```

- `exclude_unset=True` → only fields the client sent.
- `model_copy(update=...)` (Pydantic v2) merges in changes without re-validating untouched fields (use `model_validate` if you want full re-validation).

## Gotchas

- ⚠️ PUT should replace the full resource. If clients omit fields, those fields are reset to default. Use PATCH for partial updates.
- ⚠️ `exclude_defaults` ≠ `exclude_unset`. Prefer `exclude_unset` for PATCH semantics.
