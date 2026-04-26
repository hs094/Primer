# 15 — Error Handling

🔑 Raise → FastAPI serializes. Prefer domain exceptions + one handler over sprinkling `HTTPException` through services.

## Built-in `HTTPException`

```python
from fastapi import HTTPException

@app.get("/items/{id}")
async def get(id: int):
    if id not in store:
        raise HTTPException(status_code=404, detail="Item not found",
                            headers={"X-Error": "Missing"})
```

- `detail` → JSON `{"detail": ...}`. Can be any JSON-able value.

## Custom exception + handler

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class UnicornException(Exception):
    def __init__(self, name: str):
        self.name = name

@app.exception_handler(UnicornException)
async def unicorn_handler(request: Request, exc: UnicornException):
    return JSONResponse(status_code=418,
                        content={"message": f"Oops {exc.name}"})
```

## Domain-style (recommended)

```python
class AppError(Exception):
    status_code = 500
    code = "app_error"

class NotFound(AppError): status_code = 404; code = "not_found"
class Conflict(AppError): status_code = 409; code = "conflict"
class Validation(AppError): status_code = 422; code = "validation"

@app.exception_handler(AppError)
async def app_error_handler(req, exc: AppError):
    return JSONResponse(
        status_code=exc.status_code,
        content={"code": exc.code, "detail": str(exc)},
    )
```

Services raise `NotFound(...)`; HTTP concerns stay at the boundary.

## Override defaults

```python
from fastapi.exceptions import RequestValidationError
from fastapi.responses import PlainTextResponse

@app.exception_handler(RequestValidationError)
async def v_handler(req, exc):
    return PlainTextResponse(str(exc), status_code=400)
```

- `RequestValidationError` → 422 by default.
- `StarletteHTTPException` is the parent of `HTTPException`.

## Access default handlers

```python
from fastapi.exception_handlers import http_exception_handler
await http_exception_handler(request, exc)  # delegate back
```

## Gotchas

- ⚠️ Raising `HTTPException` from a service layer leaks HTTP concerns. Raise domain errors; translate at the boundary (see [[Tutorial/FastAPI 17 - Dependencies]]).
- ⚠️ `exception_handler` for `Exception` catches **everything** — place before more-specific handlers will still work because FastAPI matches by class MRO.
