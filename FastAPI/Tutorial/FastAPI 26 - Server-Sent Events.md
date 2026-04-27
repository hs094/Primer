# 26 — Server-Sent Events (SSE)

🔑 One-way streaming over HTTP using `text/event-stream`. Browsers reconnect automatically. Simpler than WebSockets when you only push server → client. First-class in FastAPI ≥ 0.135.0 via `fastapi.sse.EventSourceResponse` — no `sse-starlette` needed.

## Minimal SSE handler

```python
from collections.abc import AsyncIterable
from fastapi import FastAPI
from fastapi.sse import EventSourceResponse
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None

@app.get("/items/stream", response_class=EventSourceResponse)
async def sse_items() -> AsyncIterable[Item]:
    for i in range(10):
        yield Item(name=f"item-{i}")
```

- Set `response_class=EventSourceResponse` on the decorator and `yield` Pydantic models — each becomes a `data:` event with JSON payload.
- FastAPI sets `Cache-Control: no-cache`, `X-Accel-Buffering: no`, and pings every ~15 s automatically.

## Full event control

Yield `ServerSentEvent` to set `event` / `id` / `retry` / `comment`:

```python
from fastapi.sse import EventSourceResponse, ServerSentEvent

@app.get("/items/stream", response_class=EventSourceResponse)
async def sse_items() -> AsyncIterable[ServerSentEvent]:
    yield ServerSentEvent(comment="stream start")
    for i, item in enumerate(items):
        yield ServerSentEvent(data=item, event="item_update", id=str(i), retry=5000)
```

- `data=` accepts anything JSON-serializable (incl. Pydantic models).
- `raw_data=` for pre-formatted text (mutually exclusive with `data=`).

## Resume via Last-Event-ID

```python
from typing import Annotated
from fastapi import Header

@app.get("/items/stream", response_class=EventSourceResponse)
async def sse_items(
    last_event_id: Annotated[int | None, Header()] = None,
) -> AsyncIterable[ServerSentEvent]:
    start = last_event_id + 1 if last_event_id is not None else 0
    for i, item in enumerate(items[start:], start=start):
        yield ServerSentEvent(data=item, id=str(i))
```

## Browser side

```js
const es = new EventSource("/items/stream");
es.onmessage = e => console.log(JSON.parse(e.data));
es.addEventListener("item_update", e => ...);
```

## SSE vs WebSockets

- **SSE**: server → client only, JSON over HTTP, free reconnect, plays nicely with existing HTTP infra (proxies, auth). Works on `POST` too — useful for MCP-style protocols.
- **WebSockets** ([[Advanced/FastAPI 09 - WebSockets]]): bidirectional, binary, lower latency, custom protocols.

## Gotchas

- ⚠️ HTTP/1.1 limits ~6 connections per origin per browser tab — large dashboards run out.
- ⚠️ Cancellation: detect client disconnect via `await request.is_disconnected()` and break.
- ⚠️ If you must hand-roll SSE (older FastAPI), use `StreamingResponse(..., media_type="text/event-stream")` and emit your own `data: ...\n\n` frames — but on 0.135.0+ prefer `EventSourceResponse`.

💡 SSE is "the right tool for notifications, progress, log tails". Don't reach for WebSockets reflexively.
