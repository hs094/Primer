# CL 10 — task-queue

🔑 **One-line: inspect Task Queues, their pollers, partitions, and rate-limit/fairness config.**

A Task Queue is the named pipe Workers poll for Workflow / Activity tasks (see [[Temporal EN 06 - Workers]]). `temporal task-queue` lets you check who's polling, dispatch backlog, and tune queue-level rate limits.

## Subcommands

| Subcommand | Purpose |
|---|---|
| `describe` | Active pollers, backlog, dispatch rates |
| `list-partition` | Show partition map + matching node assignment |
| `config get` | Read queue rate-limit + fairness defaults |
| `config set` | Set queue rate-limit + fairness defaults |
| `get-build-id-reachability` | (deprecated) Build ID usability |
| `get-build-ids` | (deprecated) Compatible Build ID sets |
| `update-build-ids` | (deprecated) Add/promote Build IDs |
| `versioning` | (deprecated, replaced by Worker Deployment) Assignment + redirect rules |

## describe

```bash
temporal task-queue describe --task-queue orders
temporal task-queue describe --task-queue orders \
  --task-queue-type activity \
  --report-reachability
```

Returns currently-polling Worker identities, last poll time, backlog count, backlog age, and dispatch rate. `--report-reachability` adds a safety check used when retiring Build IDs.

## list-partition

```bash
temporal task-queue list-partition --task-queue orders
```

Shows the matching-service node owning each partition — useful when debugging hot-partition hotspots in self-hosted clusters.

## config (rate limits & fairness)

```bash
# Inspect
temporal task-queue config get \
  --task-queue orders \
  --task-queue-type activity

# Cap to 50 RPS overall, 5 RPS per fairness key
temporal task-queue config set \
  --task-queue orders \
  --task-queue-type activity \
  --queue-rps-limit 50 \
  --fairness-key-rps-limit-default 5
```

`--task-queue-type` is one of `workflow`, `activity`, `nexus`. Limits apply to dispatch, not to enqueue.

## Deprecated: Build ID versioning

The `get-build-id-reachability`, `get-build-ids`, `update-build-ids`, and `versioning` subcommands are **superseded** by Worker Deployments (see [[Temporal CL 11 - worker]] `deployment` group). They still work for legacy versioning setups.

```bash
# Legacy Build ID set (don't use for new code)
temporal task-queue update-build-ids add-new-default \
  --task-queue orders --build-id v2.1.0

# Legacy versioning rules
temporal task-queue versioning insert-assignment-rule \
  --task-queue orders --build-id v2.1.0 --percentage 25
```

## Common patterns

```bash
# "Why is the queue backing up?"
temporal task-queue describe --task-queue orders --task-queue-type activity
# -> check backlog count, oldest task age, polling worker count

# Throttle a runaway queue
temporal task-queue config set \
  --task-queue orders --task-queue-type workflow \
  --queue-rps-limit 100
```

⚠️ Build ID / `versioning` subcommands are **deprecated** in favour of Worker Deployments. New code should use `temporal worker deployment` ([[Temporal CL 11 - worker]]) — these legacy commands stop receiving new flags.

⚠️ `--task-queue-type` matters: **same name, different types** are independent queues. `orders` workflow queue and `orders` activity queue have separate backlogs and separate rate limits.

⚠️ `list-partition` is mostly a self-hosted debugging tool — output on Cloud is limited.

💡 **Takeaway:** `describe` for backlog / pollers, `config set` for RPS + fairness, skip the legacy Build ID commands in favour of Worker Deployments.
