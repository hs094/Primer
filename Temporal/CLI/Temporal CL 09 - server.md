# CL 09 — server

🔑 **One-line: run a single-process local Temporal Service for development (`start-dev`).**

`temporal server` ships exactly one user-facing subcommand — `start-dev` — which spins up Frontend + History + Matching + Worker services + Web UI in a single binary, backed by SQLite (or in-memory).

## Subcommand

| Subcommand | Purpose |
|---|---|
| `start-dev` | Launch local dev [[Temporal EN 11 - Temporal Service]] + Web UI |

## start-dev flags

| Flag | Default | Purpose |
|---|---|---|
| `--db-filename, -f` | (in-memory) | SQLite file for persistence — without it, state vanishes on restart |
| `--port, -p` | `7233` | Frontend gRPC port |
| `--ui-port` | `port + 1000` | Web UI port (default `8233`) |
| `--http-port` | — | HTTP API port |
| `--metrics-port` | — | Prometheus `/metrics` port |
| `--ip` | `127.0.0.1` | Bind address for gRPC |
| `--ui-ip` | `127.0.0.1` | Bind address for Web UI |
| `--headless` | false | Disable Web UI |
| `--namespace, -n` | (just `default`) | Pre-create extra namespaces (repeatable) |
| `--search-attribute` | — | Pre-register custom search attribute (`KEY=Type`, repeatable) |
| `--dynamic-config-value` | — | Override dynamic config (`KEY=VALUE`, repeatable) |
| `--sqlite-pragma` | — | SQLite pragma (e.g. `journal_mode=WAL`) |
| `--ui-codec-endpoint` | — | Remote payload codec for the UI |
| `--log-config` | false | Print resolved server config to stderr |

## Examples

```bash
# Bare-minimum dev loop
temporal server start-dev

# Persistent SQLite, extra namespaces, search attributes
temporal server start-dev \
  --db-filename ./temporal.db \
  --namespace orders --namespace billing \
  --search-attribute CustomerId=Keyword \
  --search-attribute OrderTotal=Double

# Headless on a CI box, expose to LAN
temporal server start-dev \
  --ip 0.0.0.0 --headless --port 7233 \
  --metrics-port 9091

# Inside Docker, reachable from host
docker run --rm \
  -p 7233:7233 -p 8233:8233 \
  temporalio/temporal server start-dev --ip 0.0.0.0
```

## Common patterns

```bash
# Hot-tweak dynamic config without restarting workers
temporal server start-dev \
  --dynamic-config-value system.forceSearchAttributesCacheRefreshOnRead=true \
  --dynamic-config-value frontend.workerVersioningRuleAPIs=true

# Use WAL for faster local writes
temporal server start-dev \
  --db-filename ./temporal.db \
  --sqlite-pragma journal_mode=WAL
```

⚠️ **Not for production.** `start-dev` is a single binary using SQLite — no replication, no scale-out, no auth. Production = self-host the real Server ([[Temporal SH 01 - Self-Hosted Overview]]) or use Temporal Cloud.

⚠️ Without `--db-filename`, all Workflow histories live in RAM. Restart = clean slate. Always persist when you have non-trivial flows in flight.

⚠️ Search Attributes registered at startup are stored in the **SQLite file**. Switching DB files = losing them — re-register on each new file.

⚠️ `--ip 0.0.0.0` with `--port 7233` exposes an unauthenticated gRPC server. Fine on a laptop firewall; bad on a public VM.

💡 **Takeaway:** `temporal server start-dev --db-filename ./db.sqlite` is the standard dev cmd; pre-register namespaces + search attributes via flags so SDK samples just work.
