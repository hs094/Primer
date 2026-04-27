# CL 11 — worker

🔑 **One-line: inspect Workers and manage Worker Deployments (the modern replacement for Task Queue Build ID versioning).**

`temporal worker` is the operational surface for [[Temporal EN 06 - Workers]]: list polling workers, describe a single worker, and — most importantly — manage **Worker Deployments**, which version-route tasks across rolling worker fleets.

## Subcommand groups

| Group | Purpose |
|---|---|
| `list` | List workers in namespace |
| `describe` | Details on one worker by `--worker-instance-key` |
| `deployment` | Full CRUD + traffic-shaping for Worker Deployments |

## list / describe

```bash
temporal worker list --query 'BuildId="v2.1.0"'
temporal worker list --limit 100

temporal worker describe \
  --worker-instance-key worker-host-7@orders@v2.1.0
```

`list` accepts a SQL-like `--query` and `--limit`.

## deployment subcommands

| Sub | Purpose |
|---|---|
| `list` | All deployments in namespace |
| `describe` | Show deployment props (`--name`, `--report-task-queue-stats`) |
| `describe-version` | Show one Version (`--deployment-name`, `--build-id`) |
| `delete` | Remove a deployment with no versions (`--name`) |
| `delete-version` | Remove a Version (`--deployment-name`, `--build-id`, `--skip-drainage`) |
| `set-current-version` | Make a Version active for new tasks |
| `set-ramping-version` | Route a `--percentage` of tasks to a Version |
| `update-version-metadata` | KV metadata on a Version |
| `manager-identity set` | Lock who can mutate a deployment |
| `manager-identity unset` | Release manager lock |

## Listing & describing

```bash
temporal worker deployment list

temporal worker deployment describe \
  --name orders --report-task-queue-stats

temporal worker deployment describe-version \
  --deployment-name orders --build-id v2.1.0
```

## Promoting a new version

```bash
# Deploy v2.1.0, then make it the current target
temporal worker deployment set-current-version \
  --deployment-name orders \
  --build-id v2.1.0 \
  --ignore-missing-task-queues   # override safety check
```

`--allow-no-pollers` overrides the requirement that the new Version actually has live workers polling.

## Canary / ramping rollout

```bash
# Send 10% of new tasks to v2.2.0 while keeping v2.1.0 as current
temporal worker deployment set-ramping-version \
  --deployment-name orders \
  --build-id v2.2.0 \
  --percentage 10

# Bump to 50%
temporal worker deployment set-ramping-version \
  --deployment-name orders \
  --build-id v2.2.0 --percentage 50

# Roll back: clear the ramp
temporal worker deployment set-ramping-version \
  --deployment-name orders --delete
```

`--unversioned` (in place of `--build-id`) routes to the unversioned worker pool.

## Metadata & manager lock

```bash
temporal worker deployment update-version-metadata \
  --deployment-name orders --build-id v2.1.0 \
  --metadata commit=abc123 --metadata releasedBy=hs094

# Take exclusive control of mutations from this user
temporal worker deployment manager-identity set \
  --deployment-name orders --self -y

# Release lock
temporal worker deployment manager-identity unset \
  --deployment-name orders -y
```

## Cleanup

```bash
temporal worker deployment delete-version \
  --deployment-name orders --build-id v2.0.0

temporal worker deployment delete --name retired-app
```

`--skip-drainage` on `delete-version` bypasses the wait for in-flight tasks to drain — destructive.

## Common patterns

```bash
# Blue-green promotion
temporal worker deployment set-current-version \
  --deployment-name orders --build-id v2.2.0 \
  --ignore-missing-task-queues

# Then prune old version once drained
temporal worker deployment delete-version \
  --deployment-name orders --build-id v2.0.0
```

⚠️ `set-current-version` flips ALL new tasks to the new Build ID immediately. For gradual rollouts use `set-ramping-version --percentage` first.

⚠️ `--skip-drainage` on `delete-version` will kill in-flight Workflow tasks targeting that Version — only use after confirming `describe-version` shows zero pollers and zero active tasks.

⚠️ `manager-identity set --self` locks every other identity out of mutating the deployment. Forgetting to `unset` leaves a deployment uneditable from CI.

💡 **Takeaway:** Worker Deployments = modern Build ID versioning. `set-current-version` for cutover, `set-ramping-version --percentage` for canary, `delete-version` once drained.
