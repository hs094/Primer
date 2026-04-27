# SH 02 — Deployment

🔑 **Four deployment shapes: Docker Compose (demo), binaries + systemd (lean), Go-embedded (custom), Helm/k8s (production). Pick by ops team, not by language.**

## Hosts Are Internal Only

⚠️ **Temporal hosts should not be exposed to the open internet.** Treat the gRPC frontend (7233) like a database port — internal LB, VPC, or VPN only. Public access goes through your application tier.

## Docker Compose (Quickstart)

Fastest path; uses `temporalio/docker-compose`:

```bash
git clone https://github.com/temporalio/docker-compose.git
cd docker-compose
docker compose up
```

Default stack: PostgreSQL + Elasticsearch + Frontend on `:7233`, Web UI on `:8080`.

⚠️ Compose default is **not production**: single-node DB, no TLS, no auth. Use only for demos / integration tests.

## Server Binaries + systemd

Two Go binaries, released independently:

- `temporalio/temporal` — server (frontend/history/matching/worker)
- `temporalio/ui-server` — Web UI

Run each under systemd. Front with Nginx/Envoy for TLS termination if you must, but keep gRPC reachable only from internal networks.

## Go Library / Embedded

Import `go.temporal.io/server/temporal` for custom plugins (auth, archiver, metrics). Requires Go 1.19+. See [[Temporal SH 03 - Embedded Server]].

## Helm Charts (Production k8s)

```bash
helm repo add temporal https://temporal.io/charts
helm install temporal temporal/temporal --version 0.73.1
```

⚠️ **Helm chart `0.73.1+` is required for Temporal Server `1.30+`.** Older charts will silently fail on schema or config keys.

Helm deploys frontend / history / matching / worker as separate Deployments and points them at *your* existing DB and Elasticsearch — chart does not provision persistence.

## Config Templating

Server YAML uses **embedded `sprig` templates** for env-var substitution (not `dockerize`). Verify before boot:

```bash
temporal-server render-config
```

Config layout: see [[Temporal RF 06 - Service Configuration]]. Dynamic toggles: [[Temporal RF 07 - Dynamic Configuration]].

## Persistence Choices

| Store | Use | Note |
|---|---|---|
| Cassandra | High write throughput, multi-DC | Default for large fleets |
| MySQL 8.0.17+ | Mid-scale | Advanced visibility supported on server 1.20+ |
| PostgreSQL 12+ | Mid-scale | Advanced visibility supported on server 1.20+ |
| SQLite | Dev / embedded only | Single-writer, `NumHistoryShards: 1` |

Visibility store is configured separately — see [[Temporal SH 09 - Self-Hosted Visibility]].

## Topology Sketch

```
clients ─► frontend (gRPC :7233) ─► history ─► persistence (Cassandra/SQL)
                  │                  ▲
                  ├─► matching ──────┤
                  └─► worker (system workflows)
                       └─► visibility store (ES / SQL)
```

Run frontend behind an internal L4/L7 LB. History and matching scale horizontally; shard count fixes history capacity.

⚠️ **`NumHistoryShards` is build-time per cluster.** Pick generously up front (recommended thousands for production); you cannot reshard without spinning a new cluster and replicating.

💡 **Takeaway:** Compose for dev, Helm for prod, binaries for lean ops. Always front with internal-only networking, template-validate config, and pin the right chart version for your server line.
