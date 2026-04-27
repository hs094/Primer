# SH 03 — Embedded Server

🔑 **Run the [[Temporal EN 11 - Temporal Service]] as an in-process Go library for tests and custom builds — never for production.**

## When To Embed

- Integration tests that need a real server (no mocks).
- Custom builds with bespoke `Authorizer`, `ClaimMapper`, archiver, or metrics handler.
- Reference implementations of the dev server (`temporal server start-dev` is itself this library).

For production, run as a separate process and point at MySQL / PostgreSQL / Cassandra — see [[Temporal SH 02 - Deployment]].

## Minimal Boot

```go
import (
    "go.temporal.io/server/temporal"
    "go.temporal.io/server/common/config"
)

server, err := temporal.NewServer(
    temporal.ForServices(temporal.DefaultServices),
    temporal.WithConfig(cfg),
    temporal.InterruptOn(temporal.InterruptCh()),
)
if err != nil {
    log.Fatal(err)
}
if err := server.Start(); err != nil {
    log.Fatal(err)
}
```

`DefaultServices` boots frontend + history + matching + worker in-process.

## Config Loading

File-based:

```go
cfg, err := config.Load(
    config.WithConfigFile("/path/to/config.yaml"),
)
```

Directory + environment:

```go
cfg, err := config.Load(
    config.WithConfigDir("./config"),
    config.WithEnv("development"),
)
```

Loader resolves `<env>.yaml` from the directory. See [[Temporal RF 06 - Service Configuration]].

## Server Options

| Option | Use |
|---|---|
| `ForServices(...)` | Pick which services run (frontend only, etc.) |
| `WithConfig(cfg)` | Pass parsed config struct |
| `WithLogger(logger)` | Plug stdlib / zap logger |
| `WithAuthorizer(a)` | Custom authorization plugin |
| `WithClaimMapper(m)` | Custom JWT claim mapping |
| `WithCustomMetricsHandler(h)` | Forward metrics to your sink |
| `InterruptOn(ch)` | Graceful shutdown signal |

See [[Temporal SH 07 - Security]] for `Authorizer` / `ClaimMapper` semantics.

## SQLite Persistence (Dev Only)

Embedded mode commonly pairs with SQLite:

```yaml
persistence:
  defaultStore: sqlite-default
  visibilityStore: sqlite-visibility
  numHistoryShards: 1
  datastores:
    sqlite-default:
      sql:
        pluginName: 'sqlite'
        databaseName: 'default'
        connectAttributes:
          mode: 'memory'
          cache: 'private'
```

Schema is bootstrapped via `sqliteschema` utilities; the dev server also pre-creates the `default` namespace.

⚠️ **SQLite footguns:**
- Single-writer — write throughput is bottlenecked.
- `mode: memory` does not survive process restart.
- Must run with `NumHistoryShards: 1`. Cannot scale.
- File-mode SQLite **does not support schema upgrades** for advanced visibility.

⚠️ **Embedded ≠ production.** No HA, no horizontal scale, no rolling upgrade story. Treat it as a test fixture.

## Reference

The Temporal CLI dev server (`temporal server start-dev`, see [[Temporal CL 09 - server]]) is the canonical embedded-server reference: SQLite setup, namespace pre-creation, port allocation. The `temporalio/samples-server` repo has examples for plugging in TLS, auth, custom metrics, and alternate datastores.

💡 **Takeaway:** Embed the server when you need a real Temporal in tests or a custom build with plugins; never run an embedded SQLite server as production infrastructure.
