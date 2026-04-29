# 03 — Workers KV

🔑 Workers KV is a globally replicated, **eventually consistent** key-value store optimised for hot reads at the edge — not for write-heavy or read-your-writes workloads.

Source: https://developers.cloudflare.com/kv/

## Shape

| Op | Latency | Consistency |
|---|---|---|
| Read (hot, cached at PoP) | <10 ms | Eventually consistent |
| Read (cold, central) | ~50–100 ms | Eventually consistent |
| Write | ~100 ms locally; **~60 s** to global propagation | Last-write-wins |

⚠️ A `put` is **not** visible globally for up to 60 seconds. Don't store auth state, balances, or anything you'll read back immediately.

## Usage

```ts
// wrangler.toml binding -> env.CACHE
const cached = await env.CACHE.get("user:42", "json");
if (cached) return Response.json(cached);

const user = await fetchUser(42);
await env.CACHE.put("user:42", JSON.stringify(user), { expirationTtl: 3600 });
```

## When to reach for KV

- 🔑 Read-mostly config, feature flags, A/B variants, rendered HTML fragments.
- 💡 CDN-style cached JSON in front of a slow origin.
- ⚠️ **Not** a database. For strong consistency see [[Edge 04 - Durable Objects]] or [[Edge 06 - D1]].

## Free tier

- 100k reads/day, 1k writes/day, 1 GB storage.
- 1 write per key per second sustained — bursty writes get rate-limited.

## 🧪 Gotchas

- Listing is paginated and slow — don't treat KV as a queryable store.
- Values up to 25 MB; keys up to 512 bytes.
- TTL is a floor, not a ceiling — entries can linger past expiry briefly.

## Tags

[[Edge]] [[Cloudflare Workers]] [[KV]] [[caching]]
