# 02 — Cloudflare Workers

🔑 A Worker is a single ES module exporting a `fetch` handler. Cloudflare runs it in a V8 isolate in the PoP closest to the request — no container, no per-instance cold start.

Source: https://developers.cloudflare.com/workers/

## Mental model

| Piece | Role |
|---|---|
| **V8 isolate** | Sandbox per Worker; many isolates share one process. ~5 ms boot. |
| **`fetch` handler** | Entry point. `(request, env, ctx) => Response`. |
| **`env`** | Bindings injected at runtime: KV, R2, D1, DO, secrets, vars. |
| **`ctx.waitUntil`** | Run async work (logging, cache fill) past the response. |
| **Wrangler CLI** | `wrangler dev` (local Miniflare), `wrangler deploy`, `wrangler tail`. |
| **`wrangler.toml`** | Declares name, entry, compat date, bindings, routes. |

## Minimal Worker

```ts
export default {
  async fetch(req: Request, env: Env, ctx: ExecutionContext) {
    return new Response("hello from the edge");
  },
};
```

## `wrangler.toml`

```toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2026-01-01"

[[kv_namespaces]]
binding = "CACHE"
id = "abc123"
```

## Why isolates matter

- 🔑 No container per tenant — one process hosts thousands of isolates. That's how cold start hits sub-ms.
- 💡 Bindings (`env.CACHE`, `env.DB`) are the only way to talk to platform services. No SDK init, no creds in code.
- ⚠️ Pricing: CPU time, not wall time. A `fetch()` to OpenAI that takes 5 s costs only the ms you spent in JS.

## Limits (free / paid)

- CPU: 10 ms / 30 s.
- Subrequests: 50 / 1000 per request.
- Script size: 1 MB / 10 MB gzipped.
- See [[Edge 09 - Edge Limits and Pitfalls]].

## Tags

[[Edge]] [[Cloudflare Workers]] [[Wrangler]] [[V8]]
