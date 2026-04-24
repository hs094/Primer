# Adv 09 — WebSockets

🔑 FastAPI uses Starlette WebSocket support. Full-duplex, not HTTP request/response.

## Hello world

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def ws_endpoint(ws: WebSocket):
    await ws.accept()
    while True:
        data = await ws.receive_text()
        await ws.send_text(f"echo: {data}")
```

## Receive / send helpers

```python
await ws.receive_text()     # str
await ws.receive_bytes()
await ws.receive_json()
await ws.send_text("...")
await ws.send_bytes(b"...")
await ws.send_json({"ok": True})
```

## Query params, headers, cookies, path params

Works the same as HTTP routes:

```python
@app.websocket("/ws/{client_id}")
async def ws(ws: WebSocket, client_id: str,
             token: Annotated[str | None, Cookie()] = None,
             q: Annotated[int | None, Query()] = None):
    ...
```

## Dependencies

Same `Depends()` mechanism works. For auth:

```python
async def get_user_from_ws(ws: WebSocket, token: Annotated[str, Cookie()]):
    user = decode_token(token)
    if not user:
        await ws.close(code=1008)        # policy violation
        raise WebSocketException(code=1008)
    return user
```

## Handling disconnects

```python
from fastapi import WebSocketDisconnect

@app.websocket("/ws")
async def ws_endpoint(ws: WebSocket):
    await ws.accept()
    try:
        while True:
            msg = await ws.receive_text()
            ...
    except WebSocketDisconnect:
        # cleanup
        ...
```

## Simple connection manager (broadcast pattern)

```python
class Manager:
    def __init__(self):
        self.active: list[WebSocket] = []
    async def connect(self, ws): await ws.accept(); self.active.append(ws)
    def disconnect(self, ws): self.active.remove(ws)
    async def broadcast(self, msg):
        for ws in self.active:
            await ws.send_text(msg)

manager = Manager()
```

⚠️ This works only within a **single** worker process. For multi-worker broadcast, use Redis pub/sub / NATS.

## Close codes (selected)

| Code | Meaning |
|---|---|
| 1000 | Normal closure |
| 1001 | Going away |
| 1008 | Policy violation |
| 1011 | Server error |
| 4000+ | Application-defined |

## Testing

```python
with client.websocket_connect("/ws") as ws:
    ws.send_text("hi")
    assert ws.receive_text() == "echo: hi"
```

## Gotchas

- ⚠️ `@app.middleware("http")` does **not** run for WS connections. Use ASGI middleware if you need it both places.
- ⚠️ Per-WS state is per-worker. Scaling horizontally requires external coordination.
- ⚠️ Avoid long CPU work inside the WS handler; it blocks the loop for that connection.
