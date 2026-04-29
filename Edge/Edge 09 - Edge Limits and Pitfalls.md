# 09 — Edge Limits and Pitfalls

🔑 The edge feels like Node until it doesn't. The constraints below kill most "just deploy our backend at the edge" plans — design around them or stay on a microVM.

## Hard constraints (V8 isolates)

| Limit | Workers / Vercel Edge |
|---|---|
| CPU per request | 10–50 ms (free), up to 30 s (paid) |
| Wall time | ~30 s typical; Vercel must start streaming in 25 s |
| Memory | ~128 MB |
| Filesystem | **None** |
| Native modules | **None** |
| Raw TCP / UDP | **None** (no `net.Socket`) |
| `child_process` | **None** |
| Long-lived connections | Awkward — see Durable Object hibernation |
| Code size | 1–10 MB gzipped |

## ⚠️ Why Postgres is hard at the edge

- 🔑 Postgres's wire protocol needs a **stateful TCP connection**. V8 isolates can't open raw sockets, and even if they could, every isolate would open its own → connection storm.
- A Worker fans out to thousands of PoPs; each opens a connection per cold start → Postgres dies at a few thousand isolates.
- Lambda hit the same wall years ago; the answer was [[PgBouncer]] / RDS Proxy in front.

## Workarounds

| Approach | How |
|---|---|
| **[[Hyperdrive]]** | Cloudflare-managed pool + edge query cache; you keep your Postgres. |
| **PgBouncer / Supavisor** | Run a pooler near the DB; Workers connect via HTTP/Postgres-over-HTTP shim. |
| **Neon / Supabase HTTP driver** | Postgres over HTTP — no socket needed, fits isolates natively. |
| **[[Edge 06 - D1]]** | Skip Postgres: SQLite at the edge with a Worker binding. |
| **Move to [[Fly.io]]** | If you need real Postgres + sockets, edge isolates are the wrong layer. |

## 🧪 Other footguns

- **Cache leakage between tenants** — module-scope state lives across requests in the same isolate. Don't cache user data in module globals.
- **`Date.now()` is coarse** — Workers freezes the clock per request to mitigate Spectre. Don't use it for benchmarking.
- **No cron in Workers proper** — use [[Cron Triggers]] (separate handler) or Durable Object alarms.
- **Subrequest limit** — 50 / 1000 outbound `fetch`es per request. RAG fan-outs hit this fast.
- **Streaming + buffering** — middleware that reads the full body kills SSE; pass through `req.body` as `ReadableStream`.

## Decision rule

- 🔑 **Read-mostly, stateless, latency-sensitive → V8 edge.**
- 🔑 **Stateful, socket-heavy, full-stdlib → microVM ([[Fly.io]]) or container.**
- 🔑 **Coordination, single-writer state → [[Durable Objects]].**

## Tags

[[Edge]] [[limits]] [[Hyperdrive]] [[PostgreSQL]] [[connection-pooling]]
