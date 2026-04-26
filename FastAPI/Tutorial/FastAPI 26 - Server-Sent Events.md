# 26 — Server-Sent Events (SSE)

🔑 One-way streaming over HTTP using `text/event-stream`. Browsers reconnect automatically. Simpler than WebSockets when you only push server → client.

## Minimal SSE handler

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio

app = FastAPI()

async def event_stream():
    n = 0
    while True:
        yield f"data: {n}\n\n"
        n += 1
        await asyncio.sleep(1)

@app.get("/events")
async def events():
    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

## SSE wire format

```
event: ping            # optional name
id: 42                 # optional, used for reconnect (Last-Event-ID)
retry: 5000            # optional, ms before reconnect
data: {"hello": 1}     # the payload

```

⚠️ Each event ends with a **blank line** (`\n\n`). Multi-line `data:` repeated lines.

## Browser side

```js
const es = new EventSource("/events");
es.onmessage = e => console.log(JSON.parse(e.data));
es.addEventListener("ping", e => ...);
```

## When SSE vs WebSockets

- **SSE**: server → client only, JSON over HTTP, free reconnect, plays nicely with existing HTTP infra (proxies, auth).
- **WebSockets** ([[Advanced/FastAPI 09 - WebSockets]]): bidirectional, binary, lower latency, custom protocols.

## Gotchas

- ⚠️ Hold-alive: send a comment `:\n\n` every ~15 s to defeat proxy idle timeouts.
- ⚠️ HTTP/1.1 limits ~6 connections per origin per browser tab — large dashboards run out.
- ⚠️ Cancellation: detect client disconnect via `await request.is_disconnected()` and break.

💡 SSE is "the right tool for notifications, progress, log tails". Don't reach for WebSockets reflexively.
