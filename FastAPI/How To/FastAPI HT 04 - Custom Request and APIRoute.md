# HT 04 — Custom Request & APIRoute

🔑 Subclass `APIRoute` to inject behavior *around* a group of routes — body decoding, body caching, custom logging — scoped to a router instead of a global middleware.

## Pattern

```python
from collections.abc import Callable
from fastapi import APIRouter, FastAPI, Request, Response
from fastapi.routing import APIRoute

class TimedRoute(APIRoute):
    def get_route_handler(self) -> Callable:
        original = super().get_route_handler()

        async def custom(request: Request) -> Response:
            from time import perf_counter
            t0 = perf_counter()
            response = await original(request)
            response.headers["X-Time-Ms"] = f"{(perf_counter()-t0)*1000:.1f}"
            return response

        return custom

router = APIRouter(route_class=TimedRoute)

@router.get("/items/")
async def items(): return [1, 2, 3]

app = FastAPI()
app.include_router(router)
```

Only routes registered on `router` get the timing header. Other routes are unaffected.

## Use cases

- Cache `request.body()` so multiple consumers can read it (logging + handler).
- Per-router logging with structured fields.
- Decrypting / decompressing request bodies before the handler sees them.
- Unified GZip-on-error handling.

## Custom Request class

```python
class GzipRequest(Request):
    async def body(self) -> bytes:
        if not hasattr(self, "_body"):
            body = await super().body()
            if "gzip" in self.headers.getlist("Content-Encoding"):
                body = gzip.decompress(body)
            self._body = body
        return self._body

class GzipRoute(APIRoute):
    def get_route_handler(self) -> Callable:
        original = super().get_route_handler()
        async def custom(request: Request) -> Response:
            return await original(GzipRequest(request.scope, request.receive))
        return custom
```

## Vs middleware

- **Middleware** — global, runs for every path.
- **Custom APIRoute** — per-router, has access to the typed handler chain.

⚠️ Custom routes still need to be wired explicitly per router (`route_class=`). They don't propagate from app-level.

💡 Reach for this when middleware is too coarse and a dependency is too narrow.
