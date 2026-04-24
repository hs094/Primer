# 19 — Middleware & CORS

🔑 Middleware = code that runs for every request, wrapping the handler. FastAPI uses Starlette's middleware.

## Custom middleware

```python
import time
from fastapi import Request

@app.middleware("http")
async def add_timing(request: Request, call_next):
    t0 = time.perf_counter()
    response = await call_next(request)
    response.headers["X-Process-Time"] = f"{time.perf_counter()-t0:.4f}"
    return response
```

- Can short-circuit by returning a `Response` without calling `call_next`.
- Run in the order they're **added**; outer middleware wraps inner.

## Class-based (Starlette / ASGI)

```python
from starlette.middleware.base import BaseHTTPMiddleware
app.add_middleware(BaseHTTPMiddleware, dispatch=my_dispatch)
```

Use `app.add_middleware(SomeMiddleware, **opts)` for ASGI-native ones (faster than `@app.middleware("http")`).

## CORS

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_origin_regex=None,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=["X-Request-Id"],
    max_age=600,
)
```

- `allow_origins=["*"]` **cannot** be combined with `allow_credentials=True`. Use regex or explicit list if you need cookies.
- OPTIONS preflight is handled automatically.

## Other useful middleware

| Middleware | Use |
|---|---|
| `TrustedHostMiddleware` | Reject requests with bad `Host` header |
| `GZipMiddleware` | gzip responses ≥ `minimum_size` |
| `HTTPSRedirectMiddleware` | Force HTTPS |
| `SessionMiddleware` (Starlette) | Signed cookie sessions |
| `ProxyHeadersMiddleware` (uvicorn) | Trust `X-Forwarded-*` behind proxy |

```python
from starlette.middleware.trustedhost import TrustedHostMiddleware
from starlette.middleware.gzip import GZipMiddleware

app.add_middleware(TrustedHostMiddleware, allowed_hosts=["example.com", "*.example.com"])
app.add_middleware(GZipMiddleware, minimum_size=1000)
```

## Gotchas

- ⚠️ Order: `app.add_middleware` adds to the **outside** — last added runs first. Think LIFO.
- ⚠️ Middleware can't access path operation params (it's below the routing layer). For per-route hooks, use `dependencies=[...]`.
- ⚠️ WebSockets are not wrapped by `@app.middleware("http")`; use ASGI middleware if you need both.
