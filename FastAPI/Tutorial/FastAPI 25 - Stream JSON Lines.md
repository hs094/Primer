# 25 — Stream JSON Lines (NDJSON)

🔑 Stream a sequence of JSON objects, one per line, to a client that consumes them as they arrive. No big array buffered, no need for the client to wait for completion.

## The handler

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import json, asyncio

app = FastAPI()

async def jsonl_rows():
    for i in range(1_000_000):
        yield json.dumps({"i": i}) + "\n"
        await asyncio.sleep(0)   # cooperative yield

@app.get("/stream")
async def stream():
    return StreamingResponse(jsonl_rows(), media_type="application/x-ndjson")
```

- `media_type="application/x-ndjson"` (or `application/jsonl`) — tells clients each line is one JSON object.
- The generator is `async` so DB/network awaits don't block the loop.

## Client side

```python
import httpx

async with httpx.AsyncClient() as c:
    async with c.stream("GET", url) as r:
        async for line in r.aiter_lines():
            obj = json.loads(line)
```

## When to use

- Large query results (DB cursors).
- Long-running batch operations with progressive results.
- LLM token-level streaming where each chunk is structured.

## Gotchas

- ⚠️ Don't `await` between `yield` for an unbounded time — proxies may close idle connections (often ~60 s).
- ⚠️ `StreamingResponse` doesn't go through `response_model` filtering — you serialize manually.
- ⚠️ If the client disconnects, the generator is cancelled — handle cleanup with `try/finally`.

💡 If the *whole* thing fits in memory, return a list. NDJSON earns its keep at scale.

See also [[FastAPI 26 - Server-Sent Events]] for event-stream framing.
