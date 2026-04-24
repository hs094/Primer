# 07 — Multiple Body Parameters

🔑 Multiple Pydantic models as params → FastAPI nests them under their parameter names.

## Two models in one body

```python
class Item(BaseModel): ...
class User(BaseModel): ...

@app.put("/items/{id}")
async def update(id: int, item: Item, user: User):
    ...
```

Request body:

```json
{ "item": { ... }, "user": { ... } }
```

## Add a singular value to the body

Use `Body()`:

```python
from fastapi import Body

async def update(
    id: int,
    item: Item,
    user: User,
    importance: Annotated[int, Body()],
):
```

Body:

```json
{ "item": {}, "user": {}, "importance": 5 }
```

## Single body param but embed

If you only have one model and still want it nested under a key:

```python
item: Annotated[Item, Body(embed=True)]
```

Body:

```json
{ "item": { ... } }
```

## Body field metadata

```python
importance: Annotated[int, Body(ge=1, le=10, examples=[3])]
```

## Gotchas

- ⚠️ With a **single** Pydantic body param, the body is that model (no outer nesting). Add `embed=True` to force nesting.
- ⚠️ Don't accidentally use `Body()` on what should be a query — it silently forces body parsing.
