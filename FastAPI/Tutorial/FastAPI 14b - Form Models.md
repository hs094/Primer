# 14b — Form Models

🔑 Group form fields into a Pydantic model — same idea as [[FastAPI 06 - Parameter Models]] but for `application/x-www-form-urlencoded` / `multipart/form-data`.

## Single model

```python
from typing import Annotated
from fastapi import FastAPI, Form
from pydantic import BaseModel

class LoginForm(BaseModel):
    username: str
    password: str

app = FastAPI()

@app.post("/login/")
async def login(data: Annotated[LoginForm, Form()]):
    return {"username": data.username}
```

Each field of the model is read as a form field.

## Forbid extra fields

```python
class LoginForm(BaseModel):
    model_config = {"extra": "forbid"}
    username: str
    password: str
```

⚠️ Without `extra="forbid"`, unexpected fields are silently dropped.

## Mix with files

Use [[FastAPI 14 - Forms and Files]] — `File()` parameters can sit alongside a `Form()` model.

## Why bother

- DRY — reuse the same shape across endpoints.
- Validation rules (`Field(min_length=…)`, `EmailStr`, etc.) come for free.
- One docs entry instead of N parameters.

🧪 If you have ≥3 form fields or any cross-field validation, switch to a Form model.
