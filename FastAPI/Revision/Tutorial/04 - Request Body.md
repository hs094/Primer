# 04 — Request Body (Pydantic)

🔑 Annotate a parameter with a Pydantic `BaseModel` → FastAPI reads, parses, and validates the JSON body.

## Model + handler

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    description: str | None = None
    tax: float | None = None

@app.post("/items/")
async def create(item: Item):
    return item
```

- Body is parsed from JSON.
- Errors → `422` with field path.
- Model is also in `/docs` schema.

## Mixed: path + query + body

```python
@app.put("/items/{item_id}")
async def update(item_id: int, item: Item, q: str | None = None):
    ...
```

FastAPI rule:
- Param in path template → **path**
- Singular type (int, str, bool) → **query**
- Pydantic model → **body**

## Using the model

```python
item_dict = item.model_dump()        # Pydantic v2
merged = {**item.model_dump(), "id": 1}
```

(Pydantic v1: `.dict()` — deprecated.)

## Without Pydantic?

- Use `dict[str, Any]` → accepted as body but no schema / validation. Rarely advisable.
- Forms: see [[14 - Forms and Files]].

## Nested

```python
class User(BaseModel):
    name: str
    addresses: list[Address]
```

Recurses into `Address` model; all nested fields validated.

## Gotchas

- ⚠️ `GET` bodies aren't forbidden but are discouraged (many tools/proxies drop them).
- ⚠️ Declaring **two** Pydantic body params changes the shape → they're nested under keys. See [[07 - Body Multiple Params]].
- 💡 Use `model_config = ConfigDict(extra="forbid")` to 422 unknown keys.
