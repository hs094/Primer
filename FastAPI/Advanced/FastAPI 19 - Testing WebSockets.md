# Adv 19 — Testing WebSockets

🔑 Use `TestClient.websocket_connect(...)` as a context manager — it returns a `WebSocketTestSession` you can `send_json` / `receive_json` against.

## Endpoint

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def ws(websocket: WebSocket):
    await websocket.accept()
    while True:
        msg = await websocket.receive_json()
        await websocket.send_json({"echo": msg})
```

## Test

```python
from fastapi.testclient import TestClient

def test_ws():
    client = TestClient(app)
    with client.websocket_connect("/ws") as ws:
        ws.send_json({"hello": "world"})
        assert ws.receive_json() == {"echo": {"hello": "world"}}
```

## API

- `send_text(s)`, `send_bytes(b)`, `send_json(obj)`
- `receive_text()`, `receive_bytes()`, `receive_json()`
- Context manager exit closes the socket.

## Headers / subprotocol

```python
with client.websocket_connect("/ws",
                              headers={"Authorization": "Bearer xyz"},
                              subprotocols=["chat.v1"]) as ws: ...
```

## Disconnects

```python
import pytest
from starlette.websockets import WebSocketDisconnect

def test_close():
    with TestClient(app).websocket_connect("/ws") as ws:
        ws.close()
        with pytest.raises(WebSocketDisconnect):
            ws.receive_json()
```

## Async tests

`TestClient` is sync. For async tests, use `httpx.AsyncClient` for HTTP and the same `TestClient` for WS — or `websockets` library directly against a real server (slower).

🧪 Always assert *both* directions: client→server, server→client.

See [[FastAPI 09 - WebSockets]] for the protocol side and [[FastAPI 12 - Async Tests]] for HTTP-async testing.
