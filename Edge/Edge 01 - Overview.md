# 01 — Overview

🔑 "Edge" runtimes trade Node's full API surface for low cold-start, geo-distributed compute. Two camps: V8 isolates (Cloudflare Workers, Vercel Edge) and microVMs (Fly.io). Pick the camp before you pick the vendor.

Source: https://developers.cloudflare.com/workers/

## V8 isolates vs microVMs vs Node containers

| Model | Cold start | Memory | APIs | Examples |
|---|---|---|---|---|
| V8 isolate | <5 ms | ~128 MB | Web standard only | Workers, Vercel Edge |
| Firecracker microVM | ~250 ms (cold), 0 ms (warm) | up to GBs | Full Linux | Fly.io, Lambda |
| Node container | seconds | GBs | Full Node | ECS, Render |

## What edge buys you

- 💡 **Geo distribution** — code runs in the PoP nearest the user, not in `us-east-1`.
- 🔑 **Sub-ms isolate boot** — V8 reuses a single process, spawns isolates per request. No container.
- ⚠️ **No native modules, no FS, no long-lived TCP** in V8 isolates. `fs`, `child_process`, raw `net` sockets — gone.
- ⚠️ **CPU cap per request** is small (10–50 ms on free tiers). Heavy work belongs in queues / Durable Objects / Node backends.

## When to use it

- 🔑 Auth, redirects, rewrites, A/B routing, header rewriting, geo personalization → edge middleware.
- 💡 Public read APIs backed by KV / cached JSON → V8 isolates.
- ⚠️ Anything that needs Postgres, long sockets, or filesystem → microVMs (Fly.io) or Node, not Workers. See [[Edge 09 - Edge Limits and Pitfalls]].

## Tags

[[Edge]] [[Cloudflare Workers]] [[Vercel]] [[Fly.io]] [[V8]] [[Firecracker]]
