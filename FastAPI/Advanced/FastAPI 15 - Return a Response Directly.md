# Adv 15 — Return a Response Directly

🔑 Whatever a route returns, FastAPI normally serializes via Pydantic + `jsonable_encoder`. Return a `Response` (or subclass) directly to skip that pipeline and control the bytes yourself.

## When you'd skip the pipeline

- Already-serialized JSON (cached bytes, third-party payload).
- Custom content types (`application/xml`, binary).
- Custom headers/cookies tied to the body.
- Performance — bypass field filtering on hot paths.

## Example: pre-serialized JSON

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse
from fastapi.encoders import jsonable_encoder
from datetime import datetime

app = FastAPI()

@app.get("/items/")
async def f():
    payload = {"created": datetime.now()}
    return JSONResponse(content=jsonable_encoder(payload))
```

`jsonable_encoder` converts non-JSON-native types (datetime, UUID, Pydantic models) to native ones — handy when you build a `Response` manually.

## Example: arbitrary bytes / content type

```python
from fastapi import Response

@app.get("/feed.xml")
async def rss() -> Response:
    return Response(content=build_rss(), media_type="application/xml")
```

## What's bypassed

When you return a `Response` directly:
- `response_model` filtering — **off**.
- Default `JSONResponse` encoding — **off**.
- OpenAPI inference of schema — degrades; declare via `responses=`.

⚠️ The handler's return annotation no longer drives the response shape — your `Response` content is final.

## Tip — keep docs accurate

```python
@app.get("/feed.xml",
         response_class=Response,
         responses={200: {"content": {"application/xml": {}}}})
```

`response_class` tells docs the default content type.

💡 Default to "let FastAPI do it". Reach for `Response` only when you need raw bytes or non-JSON content.
