# HT 11 — Use Old 403 Authentication Status Codes

🔑 Modern FastAPI returns **401 Unauthorized** when credentials are missing/invalid (RFC-correct). Older versions returned **403 Forbidden**. Pin the legacy behavior if you have ancient clients that distinguish wrongly.

## Background

- **401 Unauthorized** = "you are not authenticated" (please provide credentials).
- **403 Forbidden** = "you are authenticated but not allowed".

Older FastAPI / Starlette mistakenly used 403 for missing auth. Most clients didn't care; some did.

## Restoring 403

```python
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="token",
    auto_error=False,        # disable the default 401
)

async def current_user(token: str | None = Depends(oauth2_scheme)):
    if not token:
        raise HTTPException(403, "Not authenticated")
    return decode(token)
```

`auto_error=False` lets you decide the status code yourself.

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

If you keep 401, also include:

```python
raise HTTPException(401,
                    "Not authenticated",
                    headers={"WWW-Authenticate": "Bearer"})
```

Required by RFC 7235; some clients skip retries without it.

💡 Don't break the spec for the present convenience of buggy clients — version your way out.
