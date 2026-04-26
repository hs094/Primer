# HT 07 — Separate OpenAPI Schemas for Input vs Output

🔑 By default, FastAPI generates **two** schemas per Pydantic model — `Item` (input) and `Item-Output` (output) — because input may have defaults the output never omits. You can disable this and merge to one if your models match.

## Default behavior (recommended)

Input model with defaulted fields:

```python
class Item(BaseModel):
    name: str
    description: str | None = None       # optional on input
    tax: float = 10.5                    # default on input
```

OpenAPI shows:
- `Item` — POST request body, with `description` & `tax` optional.
- `Item-Output` — response, where `description` & `tax` are *required* (always populated).

Why: a server that always returns `tax=10.5` is a stronger contract than the input's "may default to 10.5".

## Disable separation

```python
app = FastAPI(separate_input_output_schemas=False)
```

Now both directions share `Item`. Use only when input/output really are identical *and* clients can handle the looser contract.

## Per-route response shapes

Often the cleaner answer: distinct models.

```python
class ItemIn(BaseModel):
    name: str
    description: str | None = None

class ItemOut(BaseModel):
    id: int
    name: str
    description: str | None
    created_at: datetime

@app.post("/items/", response_model=ItemOut)
async def create(item: ItemIn) -> ItemOut: ...
```

This is usually the right design, not a workaround.

## Why it matters

Generated SDKs use the OpenAPI types directly. If `description` is optional on output but the server *always* fills it, mobile clients write null-safety code for nothing. The default split saves them.

💡 Keep the split (default). Override only when you've audited and the contract truly is symmetric.
