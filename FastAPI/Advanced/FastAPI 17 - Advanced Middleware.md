# Adv 17 — Advanced Middleware

🔑 Middleware = ASGI wrappers that run before/after every request. FastAPI ships with several from Starlette; add custom ones via `app.add_middleware(...)`.

## Built-ins

```python
from fastapi import FastAPI
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI()
app.add_middleware(HTTPSRedirectMiddleware)
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["example.com", "*.example.com"])
app.add_middleware(GZipMiddleware, minimum_size=1000, compresslevel=5)
```

- **HTTPSRedirect** — 308 to `https://`. Behind a TLS-terminating proxy, configure `proxy_headers` instead.
- **TrustedHost** — drops requests with unexpected `Host` headers (defense vs Host header attacks).
- **GZip** — compresses responses ≥ minimum_size.

## CORS (recap)

```python
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(CORSMiddleware,
                   allow_origins=["https://app.example.com"],
                   allow_credentials=True,
                   allow_methods=["*"],
                   allow_headers=["*"])
```

See [[FastAPI 19 - Middleware and CORS]].

## Custom middleware (function form)

```python
from fastapi import Request
from time import perf_counter

@app.middleware("http")
async def timing(request: Request, call_next):
    t0 = perf_counter()
    resp = await call_next(request)
    resp.headers["X-Time-Ms"] = f"{(perf_counter()-t0)*1000:.1f}"
    return resp
```

## Order matters

Middleware applied with `add_middleware` is executed **outside-in**: the *last added* runs *first*. Plan stacking deliberately (e.g. CORS outermost, GZip inner).

## Watch out

- ⚠️ Middleware sees raw bytes; consuming `request.body()` here breaks downstream parsing unless you cache it.
- ⚠️ Streaming responses + GZip = potential buffering. Test with realistic payload sizes.
- ⚠️ Don't put auth in middleware — use [[FastAPI 17 - Dependencies]] / `Depends(get_current_user)` so the path-op signature is honest.

💡 Middleware is for cross-cutting concerns (timing, headers, compression). Business rules belong in dependencies or services.
