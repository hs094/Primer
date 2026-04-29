# Edge Runtimes Knowledge Pack

Crisp, scannable revision notes covering edge compute platforms — Cloudflare Workers, Vercel Edge, Fly.io — and the storage/coordination primitives that go with them.

## How to use

- Each note = one concept, optimised for recall.
- Code blocks are the minimum that exercises the feature.
- 🔑 = core idea. ⚠️ = footgun. 💡 = nuance. 🧪 = sharp edge.
- Source URL on each note.

## Notes

| # | Topic | Note |
|---|---|---|
| 01 | Edge runtimes vs Node — V8 isolates, microVMs, when to use | [[Edge 01 - Overview]] |
| 02 | Cloudflare Workers — V8 isolates, Wrangler, `fetch` handler | [[Edge 02 - Cloudflare Workers]] |
| 03 | Workers KV — eventually consistent global KV | [[Edge 03 - Workers KV]] |
| 04 | Durable Objects — strong consistency, coordination, hibernation | [[Edge 04 - Durable Objects]] |
| 05 | R2 — S3-compatible object storage, zero egress | [[Edge 05 - R2]] |
| 06 | D1 — SQLite at the edge, vs Hyperdrive | [[Edge 06 - D1]] |
| 07 | Fly.io — Firecracker microVMs, regions, volumes | [[Edge 07 - Fly.io]] |
| 08 | Vercel Edge — Edge Functions vs Middleware, runtime constraints | [[Edge 08 - Vercel Edge]] |
| 09 | Edge limits & pitfalls — Postgres at the edge, workarounds | [[Edge 09 - Edge Limits and Pitfalls]] |

## Quick decision matrix

| Need | Reach for |
|---|---|
| Auth / redirects / geo routing | [[Edge 08 - Vercel Edge]] middleware or [[Edge 02 - Cloudflare Workers]] |
| Read-mostly cached JSON | [[Edge 03 - Workers KV]] |
| Single-writer state, chat rooms | [[Edge 04 - Durable Objects]] |
| User-uploaded media | [[Edge 05 - R2]] |
| Edge SQL, new app | [[Edge 06 - D1]] |
| Existing Postgres / sockets / native deps | [[Edge 07 - Fly.io]] |
| Existing Postgres, want edge | [[Hyperdrive]] (see [[Edge 09 - Edge Limits and Pitfalls]]) |

## Tags

[[Edge]] [[Cloudflare Workers]] [[Durable Objects]] [[R2]] [[D1]] [[Fly.io]] [[Vercel]] [[V8]] [[Firecracker]] [[Hyperdrive]]
