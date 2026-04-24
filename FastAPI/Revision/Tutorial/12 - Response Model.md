# 12 — Response Model

🔑 `response_model=` (or return type) declares the *output* schema. FastAPI **validates, filters, and documents** the response.

## Two ways to declare

```python
# Via decorator kwarg
@app.post("/items/", response_model=Item)
async def create(item: ItemIn):
    return item

# Via return annotation (same effect, preferred)
@app.post("/items/")
async def create(item: ItemIn) -> Item:
    return item
```

`response_model=` wins when both are set.

## Why use separate in/out models

```python
class UserIn(BaseModel):
    username: str
    password: str       # sensitive
    email: EmailStr

class UserOut(BaseModel):
    username: str
    email: EmailStr     # password stripped

@app.post("/users/", response_model=UserOut)
async def create(user: UserIn) -> Any:
    return user         # password filtered out on the way out
```

## Filtering flags

| Flag | Effect |
|---|---|
| `response_model_exclude_unset=True` | Omit fields that were never set (keeps explicit `None`) |
| `response_model_exclude_defaults=True` | Omit fields that equal defaults |
| `response_model_exclude_none=True` | Omit `None` fields |
| `response_model_include={"a","b"}` | Whitelist |
| `response_model_exclude={"password"}` | Blacklist |

## Inheritance pattern

```python
class UserBase(BaseModel):
    username: str
    email: EmailStr

class UserIn(UserBase):
    password: str

class UserOut(UserBase):
    id: int
```

## Return type bypass (rare)

When the object isn't a valid `response_model` (e.g., returning a `Response` directly):

```python
@app.get("/legacy", response_model=None)
def f() -> Response | dict:
    ...
```

## Gotchas

- ⚠️ Without `response_model`, the raw return is still serialized, but **no filtering/validation** happens.
- ⚠️ `response_model_exclude_unset` respects "was it set?" — Pydantic v2 tracks this via `model_fields_set`.
- 💡 Use input / output / DB models triad for non-trivial entities (avoid leaking internal fields).
