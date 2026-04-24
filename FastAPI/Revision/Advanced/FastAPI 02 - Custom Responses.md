# Adv 02 — Custom Response Classes

🔑 Override how the response is serialized. Default is `JSONResponse`.

## Built-in classes

| Class | Content-Type | Use |
|---|---|---|
| `JSONResponse` | `application/json` | default |
| `ORJSONResponse` | `application/json` | `orjson` — fastest; needs `orjson` |
| `UJSONResponse` | `application/json` | `ujson`; less strict |
| `HTMLResponse` | `text/html` | templates |
| `PlainTextResponse` | `text/plain` | logs, health checks |
| `RedirectResponse` | — | 307 redirect (change `status_code=`) |
| `StreamingResponse` | any | chunked streaming |
| `FileResponse` | by extension | send a file from disk |

## Setting per-route

```python
from fastapi.responses import HTMLResponse

@app.get("/", response_class=HTMLResponse)
async def home():
    return "<h1>Hi</h1>"
```

- `response_class` also sets the default rendering in `/docs`.

## Global default

```python
app = FastAPI(default_response_class=ORJSONResponse)
```

## Returning a response directly

```python
from fastapi.responses import JSONResponse

@app.get("/x")
async def f():
    return JSONResponse(status_code=201, content={"ok": True},
                        headers={"X-Custom": "1"})
```

When the return value is already a `Response`, FastAPI doesn't validate against `response_model`. Use `response_model=None` if your type confuses inference.

## Streaming

```python
from fastapi.responses import StreamingResponse

def gen():
    for chunk in big_chunks():
        yield chunk

@app.get("/stream")
def stream():
    return StreamingResponse(gen(), media_type="text/plain")
```

- Generator can be sync or async.
- Don't buffer the whole stream.

## FileResponse

```python
from fastapi.responses import FileResponse
return FileResponse("report.pdf", filename="report.pdf",
                    media_type="application/pdf")
```

Handles `Range` requests and ETag / Last-Modified automatically.

## RedirectResponse

```python
from fastapi.responses import RedirectResponse
return RedirectResponse("/items/1", status_code=status.HTTP_303_SEE_OTHER)
```

## Gotchas

- ⚠️ When returning a bare `Response`, the `response_model` is ignored (no filtering).
- ⚠️ `ORJSONResponse` serializes `datetime` as ISO strings — consistent but with microseconds; normalize if clients are picky.
