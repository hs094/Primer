# 04 — Durable Objects

🔑 A Durable Object is a **single-instance, strongly-consistent** Worker tied to a name. Every request for the same `id` lands on the same isolate, in one location, with its own transactional storage. The coordination primitive Workers were missing.

Source: https://developers.cloudflare.com/durable-objects/

## What's different from a Worker

| | Worker | Durable Object |
|---|---|---|
| Instances | Many (one per PoP, per request) | **One per id, globally** |
| State | Stateless | In-memory **and** persistent storage |
| Consistency | None | **Strong**, serializable |
| Use | Stateless API | Coordination, sessions, rooms, counters |

## Minimal class

```ts
export class Counter {
  constructor(private state: DurableObjectState) {}
  async fetch(req: Request) {
    let n = (await this.state.storage.get<number>("n")) ?? 0;
    n++;
    await this.state.storage.put("n", n);
    return Response.json({ n });
  }
}
// Worker: const id = env.COUNTER.idFromName("global"); env.COUNTER.get(id).fetch(req)
```

## Storage API

- 🔑 `state.storage.get/put/delete` — transactional, serializable, on disk colocated with the object.
- 💡 `state.blockConcurrencyWhile(async () => …)` — serialize init logic.
- `transaction(cb)` — multi-key atomicity.

## WebSocket hibernation

- ⚠️ Long-lived WebSockets normally pin compute. **Hibernation** lets the DO be evicted while sockets remain open; it wakes only on a message.
- Use `state.acceptWebSocket(ws)` + `webSocketMessage` handler instead of in-memory `addEventListener`.
- 🧪 Critical for chat / multiplayer at scale — otherwise idle rooms cost compute.

## When to use

- 🔑 Chat rooms, multiplayer game state, collaborative editing (CRDT host).
- Per-user rate limiting, per-tenant counters.
- AI agent session state, conversation memory pinned to a user id.

## Tags

[[Edge]] [[Cloudflare Workers]] [[Durable Objects]] [[WebSocket]] [[coordination]]
