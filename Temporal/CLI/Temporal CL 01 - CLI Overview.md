# CL 01 — CLI Overview

🔑 **One-line: terminal access to a Temporal Service for managing, monitoring, and debugging Workflows, Activities, and the Service itself.**

The `temporal` binary is the single entry point for everything outside SDK code: starting Workflows, signalling them, registering Namespaces, running a dev server, and inspecting Task Queues. It speaks gRPC to the Frontend Service.

## Command groups

| Group | Purpose |
|---|---|
| `activity` | Manage standalone or workflow-invoked Activities ([[Temporal CL 03 - activity]]) |
| `batch` | Inspect or terminate batch jobs ([[Temporal CL 04 - batch]]) |
| `config` | Read/write profile config in `temporal.toml` ([[Temporal CL 05 - config]]) |
| `env` | Read/write environment presets in `temporal.yaml` ([[Temporal CL 06 - env]]) |
| `operator` | Cluster/Namespace/Search-Attribute/Nexus admin ([[Temporal CL 07 - operator]]) |
| `schedule` | Manage [[Temporal DV 08 - Schedules]] ([[Temporal CL 08 - schedule]]) |
| `server` | Run a local dev [[Temporal EN 11 - Temporal Service]] ([[Temporal CL 09 - server]]) |
| `task-queue` | Describe/configure Task Queues ([[Temporal CL 10 - task-queue]]) |
| `worker` | Inspect Workers and Worker Deployments ([[Temporal CL 11 - worker]]) |
| `workflow` | Start/signal/query/terminate Workflows ([[Temporal CL 12 - workflow]]) |

## Global flags

These apply to nearly every subcommand — set once via `temporal env set` or pass per-call:

```bash
temporal workflow list \
  --address my-service.aws.api.temporal.io:7233 \
  --namespace prod \
  --tls \
  --tls-cert-path ./client.pem --tls-key-path ./client.key \
  --output json
```

| Flag | Purpose |
|---|---|
| `--address` | Frontend gRPC host:port (default `localhost:7233`) |
| `--namespace, -n` | Target [[Temporal EN 12 - Namespaces]] (default `default`) |
| `--api-key` | API key (Cloud / authenticated services) |
| `--tls`, `--tls-cert-path`, `--tls-key-path`, `--tls-ca-path` | mTLS material |
| `--tls-disable-host-verification` | Skip hostname check |
| `--env` | Active environment profile (default `default`) |
| `--output, -o` | `text`, `json`, `jsonl`, `none` |
| `--time-format` | `relative`, `iso`, `raw` |
| `--log-level` | `debug`, `info`, `warn`, `error`, `never` |
| `--codec-endpoint` | Remote payload codec for decoding |
| `--grpc-meta` | Extra gRPC headers (`KEY=VALUE`) |
| `--command-timeout` | Per-call timeout (`0s` = none) |

## Environment variables

The CLI honours these overrides — handy for CI:

| Var | Maps to |
|---|---|
| `TEMPORAL_ADDRESS` | `--address` |
| `TEMPORAL_NAMESPACE` | `--namespace` |
| `TEMPORAL_API_KEY` | `--api-key` |
| `TEMPORAL_TLS_CERT` / `TEMPORAL_TLS_KEY` / `TEMPORAL_TLS_CA` | TLS paths |
| `HTTPS_PROXY` | Outbound proxy |

## Common patterns

```bash
# Quick local dev loop
temporal server start-dev
temporal workflow start --task-queue hello --type Greeting --workflow-id wf-1
temporal workflow list

# Connect to Temporal Cloud
temporal env set --env prod --key address --value my-ns.tmprl.cloud:7233
temporal env set --env prod --key namespace --value my-ns.123
temporal env set --env prod --key tls-cert-path --value ./client.pem
temporal env set --env prod --key tls-key-path  --value ./client.key
temporal --env prod workflow list
```

⚠️ The CLI is a thin wrapper over the gRPC API — same RPC layer the [[Temporal DV 16 - Temporal Client]] uses. CLI calls are auditable but not transactional; pair with `--query` filters and `--yes` only after dry-running.

⚠️ Output format `text` is for humans only. Anything you parse should use `--output json` (or `jsonl`) — the text columns change between releases.

💡 **Takeaway:** `temporal` is the ops-side equivalent of the SDK Client; flags compose the same way as RPC arguments — see [[Temporal RF 02 - Commands]] for the underlying primitives.
