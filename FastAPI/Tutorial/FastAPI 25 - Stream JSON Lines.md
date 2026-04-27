# 25 — Stream JSON Lines (NDJSON)

🔑 Stream a sequence of JSON objects, one per line, to a client that consumes them as they arrive. No big array buffered, no need for the client to wait for completion. First-class in FastAPI ≥ 0.134.0 — just `yield` from the path op.

## The handler

```python
from collections.abc import AsyncIterable
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None

@app.get("/items/stream")
async def stream_items() -> AsyncIterable[Item]:
    for i in range(1_000_000):
        yield Item(name=f"item-{i}", description=None)
```

- No `StreamingResponse`, no manual `json.dumps` — `yield` from the path op and FastAPI emits one JSON object per line.
- Response media type is **`application/jsonl`**.
- The return-type annotation (`AsyncIterable[Item]`) drives Pydantic's Rust serializer for each yielded item.
- Sync version works too — annotate `-> Iterable[Item]` and use plain `def`; FastAPI runs it on the threadpool.

## Output on the wire

```
{"name":"item-0","description":null}
{"name":"item-1","description":null}
...
```

Not a JSON array — each line is a complete JSON object (NDJSON).

## Client side

```python
import httpx, json

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
- ⚠️ Streaming responses skip `response_model` filtering of the *response shape* — the return-type annotation governs per-item serialization instead.
- ⚠️ If the client disconnects, the generator is cancelled — handle cleanup with `try/finally`.

💡 If the *whole* thing fits in memory, return a list. NDJSON earns its keep at scale.

See also [[FastAPI 26 - Server-Sent Events]] for event-stream framing, and `advanced/stream-data/` for raw bytes/strings via `StreamingResponse` with `yield`.
