# Adv 13 — Stream Data

🔑 Stream large or open-ended payloads with `StreamingResponse`. The body is an (async) iterable — each chunk goes out as it's yielded.

## Async generator (preferred)

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

async def gen():
    for chunk in source():        # async DB cursor, file read, LLM tokens
        yield chunk               # bytes or str

@app.get("/stream")
async def stream():
    return StreamingResponse(gen(), media_type="text/plain")
```

## Stream a file without loading

```python
def file_iter(path: str, chunk: int = 8192):
    with open(path, "rb") as f:
        while data := f.read(chunk):
            yield data

return StreamingResponse(file_iter("big.bin"), media_type="application/octet-stream")
```

For static files prefer [[FastAPI 02 - Custom Responses]]'s `FileResponse`. Use streaming for *generated* bytes.

## Sync generator inside async app

OK — FastAPI runs sync generators on a threadpool. But ⚠️ if `gen()` does blocking I/O *inside* an `async def` route, it stalls the loop. Either:
- Make the generator `async`, or
- Define the route as `def` so it runs in the threadpool.

## Detect disconnect

```python
@app.get("/stream")
async def stream(request: Request):
    async def gen():
        try:
            for x in source():
                if await request.is_disconnected():
                    break
                yield x
        finally:
            cleanup()
    return StreamingResponse(gen())
```

⚠️ Without this check, a closed client may leave you computing forever.

## Headers / status

```python
StreamingResponse(gen(), status_code=200, headers={"X-Job": "42"})
```

## Compared to NDJSON / SSE

- Plain `StreamingResponse` = bytes pipe, you set framing.
- [[FastAPI 25 - Stream JSON Lines]] = NDJSON convention.
- [[FastAPI 26 - Server-Sent Events]] = SSE framing for browsers.

💡 Pick the framing first; use `StreamingResponse` for everything that doesn't fit a fixed schema.
