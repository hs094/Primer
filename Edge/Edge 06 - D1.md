# 06 — D1

🔑 D1 is **SQLite running on Cloudflare's edge**, accessed via a Worker binding. One primary region, read replicas in others — picks the right replica per request.

Source: https://developers.cloudflare.com/d1/

## Shape

| | D1 |
|---|---|
| Engine | SQLite |
| Access | Worker binding only (`env.DB`) — no public TCP |
| Writes | Single primary region |
| Reads | Replicated to nearby regions ("Sessions API" pins to a snapshot) |
| Size | 10 GB / DB on paid plan |

## Usage

```ts
const { results } = await env.DB
  .prepare("select id, email from users where id = ?")
  .bind(userId)
  .all();
```

## Why it exists

- 🔑 Workers can't open raw TCP to Postgres without [[Hyperdrive]] or workarounds — see [[Edge 09 - Edge Limits and Pitfalls]].
- 💡 D1 sidesteps that: SQLite + a binding, no driver, no pool, no TLS dance.
- 🧪 Built on the same storage layer as [[Durable Objects]] under the hood.

## D1 vs Hyperdrive

| | D1 | Hyperdrive |
|---|---|---|
| Engine | SQLite (managed) | Your Postgres / MySQL |
| Latency | Read-replica close to user | Pooled connections + cache to your DB |
| Migration | New schema | Lift-and-shift existing app |
| Best for | New, edge-first apps | Existing Postgres apps moved to Workers |

## ⚠️ Limits

- SQLite semantics — no `LATERAL`, limited window funcs, no extensions.
- Writes are a single region; cross-region writes pay round-trip.
- 100 concurrent statements per query; rows per query capped.

## Migrations

- `wrangler d1 migrations create`, `wrangler d1 migrations apply`.
- Plain `.sql` files. No Alembic / Prisma magic — keep it simple.

## Tags

[[Edge]] [[D1]] [[SQLite]] [[Cloudflare Workers]] [[Hyperdrive]]
