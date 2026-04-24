# 08 — Body `Field()` & Nested Models

🔑 `Field()` inside a Pydantic model = `Query()`/`Path()`/`Body()` but for model attributes.

## `Field()` usage

```python
from pydantic import BaseModel, Field

class Item(BaseModel):
    name: str = Field(..., min_length=1, max_length=100, examples=["Thor"])
    price: float = Field(gt=0, description="Must be positive")
    tags: list[str] = Field(default_factory=list)
    tax: float | None = Field(default=None, ge=0)
```

Options mirror `Query()` — length, regex/pattern, numeric bounds, description, examples, alias, `deprecated`.

## Nested models

```python
class Image(BaseModel):
    url: HttpUrl
    name: str

class Item(BaseModel):
    images: list[Image] | None = None
```

- Validates each nested object.
- Recurses arbitrarily deep.

## Pure list / set / dict bodies

```python
async def f(items: list[Item]):
    ...                         # body = [ {...}, {...} ]

async def f(tags: set[str]):    # deduplicated
    ...

async def f(scores: dict[str, float]):
    ...                         # keys validated as str
```

## Special types (via Pydantic)

- `HttpUrl`, `EmailStr` (needs `email-validator`), `AnyUrl`, `IPvAnyAddress`, `SecretStr`.
- `datetime`, `date`, `time`, `timedelta`, `UUID`, `Decimal`.
- `Literal["a", "b"]`, `Enum`, `Annotated` for constraints.

## Gotchas

- ⚠️ Mutable default args: use `Field(default_factory=list)`, not `= []`.
- ⚠️ `Field(gt=0)` needs an explicit default `= Field(...)` if the field is required.
- 💡 Pydantic v2: prefer `model_config = ConfigDict(...)` for model-level behavior.
