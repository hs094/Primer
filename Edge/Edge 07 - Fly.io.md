# 07 — Fly.io

🔑 Fly runs your container as a **Firecracker microVM** ("Machine") in regions you pick. Full Linux, persistent disks, real Postgres — the antidote to V8-isolate constraints.

Source: https://fly.io/docs/

## Mental model

| Piece | Role |
|---|---|
| **Machine** | Firecracker microVM. Boots in ~250 ms cold, instantly when warm. |
| **App** | Group of Machines sharing a `fly.toml` and DNS name. |
| **Region** | 3-letter code (`iad`, `fra`, `sin`). Machines live in one region each. |
| **Volume** | Per-Machine persistent disk (`fly volumes create`). |
| **flyctl** | CLI: `fly launch`, `fly deploy`, `fly scale`, `fly ssh console`. |

## `fly.toml` essentials

```toml
app = "my-api"
primary_region = "iad"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 0

[[vm]]
  size = "shared-cpu-1x"
  memory = "512mb"
```

## What's good

- 🔑 **Real OS** — Postgres clients, raw TCP, FS, native modules, GPUs all work.
- 💡 **Auto-stop / auto-start** — idle Machines stop to $0; first request boots one in ~250 ms. Edge-ish economics, container power.
- Fly Postgres / Managed Postgres clusters in same region — proper connection pooling, no edge gymnastics.
- Anycast IPs route users to the nearest Machine.

## ⚠️ vs Workers

- Cold start is ~250 ms (Firecracker), not <5 ms (V8). Fine for APIs, painful for super-bursty traffic.
- No KV / DO equivalent — bring your own Redis / Postgres.
- Pay-per-Machine-second, not per-request.

## When to pick Fly

- 🔑 You need Postgres + persistent state + a single global URL, not edge KV.
- Long-running connections (WebSockets, SSE, gRPC) without DO hibernation gymnastics.
- Python / Go / Rust services that don't fit in a V8 isolate.

## Tags

[[Edge]] [[Fly.io]] [[Firecracker]] [[microVM]] [[PostgreSQL]]
