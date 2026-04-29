# 04 — Request / Reply

🔑 RPC-style request/reply is built on pub/sub: the requester subscribes to a private inbox, the responder publishes back to it.

Source: https://docs.nats.io/nats-concepts/core-nats/reqreply

## How It Works
1. Client generates a unique **inbox** subject (e.g. `_INBOX.abc123`).
2. Client `PUB`s to `service.foo` with `reply-to = _INBOX.abc123`.
3. Server-side handler `PUB`s the response to `_INBOX.abc123`.
4. Client receives, returns from `request()`.

The inbox subject is just a normal subject — no special server feature.

## Python
```python
# Client
try:
    resp = await nc.request("math.add", b'{"a": 2, "b": 3}', timeout=1.0)
    print(resp.data)  # b'5'
except nats.errors.NoRespondersError:
    print("no service available")
except asyncio.TimeoutError:
    print("timed out")

# Service
async def add(msg) -> None:
    await msg.respond(b"5")

await nc.subscribe("math.add", cb=add)
```

## Patterns
- ⚠️ **Always pass a timeout** — without it a missing responder hangs forever.
- 💡 **No-responders fast-fail** — server returns a 503-ish empty message immediately when nobody is subscribed.
- 🧪 **Scatter-gather**: subscribe to a temporary inbox manually, publish, collect N responses for time-window T.
- Multiple responders + a [[Queue Group]] → automatic load balancing.

## Tags
[[NATS]] [[Pub Sub]] [[Messaging]]
