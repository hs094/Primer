# HT 11 — Use Old 403 Authentication Status Codes

🔑 FastAPI ≥ 0.122 returns **401 Unauthorized** with a `WWW-Authenticate` header when credentials are missing (RFC 7235 / 9110). Earlier versions returned **403 Forbidden**. Pin the legacy 403 by subclassing the security class and overriding `make_not_authenticated_error()`.

## Background

- **401 Unauthorized** = "you are not authenticated" (please provide credentials).
- **403 Forbidden** = "you are authenticated but not allowed".

Pre-0.122 FastAPI raised 403 for missing auth. Most clients didn't care; some did.

## Restoring 403

```python
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer

class HTTPBearer403(HTTPBearer):
    def make_not_authenticated_error(self) -> HTTPException:
        return HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not authenticated",
        )

CredentialsDep = Annotated[HTTPAuthorizationCredentials, Depends(HTTPBearer403())]

app = FastAPI()

@app.get("/me")
def read_me(credentials: CredentialsDep):
    return {"token": credentials.credentials}
```

`make_not_authenticated_error()` returns the `HTTPException` instance (FastAPI raises it). The same override works on `OAuth2PasswordBearer`, `APIKeyHeader`, etc.

## When to actually do this

- ✅ Long-tail mobile clients hardcoded to retry only on 403.
- ✅ Migration window from a legacy gateway that mapped 401 → logout-loop.
- ❌ Greenfield APIs — use 401, that's the spec.

## Cleaner: deprecate per-client

Instead of weakening the whole API, route legacy clients through a versioned prefix:

```python
api_v1 = APIRouter(prefix="/v1")          # returns 403 for missing auth (legacy)
api_v2 = APIRouter(prefix="/v2")          # returns 401 (modern)
```

Then sunset v1.

## WWW-Authenticate header

If you keep 401 (the default), FastAPI's built-in security classes already attach a sensible `WWW-Authenticate` header. When raising your own 401 from a handler, set it explicitly:

```python
raise HTTPException(
    status.HTTP_401_UNAUTHORIZED,
    "Not authenticated",
    headers={"WWW-Authenticate": "Bearer"},
)
```

Required by RFC 7235; some clients skip retries without it.

💡 Don't break the spec for the present convenience of buggy clients — version your way out.
