# CL 12 — workflow

🔑 **One-line: full lifecycle control of [[Temporal EN 03 - Workflows]] — start, signal, query, update, cancel, terminate, reset, delete.**

The biggest command group. Roughly four buckets: **start** (`start`, `execute`, `signal-with-start`, `start-update-with-start`, `execute-update-with-start`), **inspect** (`list`, `count`, `describe`, `show`, `metadata`, `result`, `trace`, `stack`), **interact** (`signal`, `query`, `update`, `pause`/`unpause`), and **end** (`cancel`, `terminate`, `delete`, `reset`, `update-options`).

## Subcommands

| Subcommand | Purpose |
|---|---|
| `start` | Start a Workflow, return Workflow + Run IDs |
| `execute` | Start and block until done, stream output to stdout |
| `signal-with-start` | Signal an existing run, or start + signal |
| `start-update-with-start` | Update existing run, or start + update (wait for accept/reject) |
| `execute-update-with-start` | Same, but wait for update completion |
| `describe` | Static details for one Workflow Execution |
| `metadata` | User-set summary/details (UI memo) |
| `list` | List Workflows (`--query`, `--archived`, `--limit`) |
| `count` | Count matches for a `--query` |
| `show` | Print Event History (`--detailed`, `--follow`) |
| `result` | Block until result, print it |
| `trace` | Live tree of executions + children |
| `stack` | `__stack_trace` query — live thread dump |
| `query` | Run a custom Query type |
| `signal` | Send a Signal (single or `--query` bulk) |
| `update` | `start` / `execute` / `result` / `describe` Workflow Updates |
| `pause` | Pause execution (experimental) |
| `unpause` | Resume execution |
| `cancel` | Send cancellation request (graceful) |
| `terminate` | Force end (no cleanup) |
| `delete` | Delete execution + history |
| `reset` | Rewind to a prior event |
| `update-options` | Mutate versioning / behaviour overrides |
| `fix-history-json` | Reserialize a history JSON file |

## Starting Workflows

```bash
# Fire and forget
temporal workflow start \
  --workflow-id order-42 \
  --type OrderWorkflow \
  --task-queue orders \
  --input '{"customer":"abc","total":4200}' \
  --search-attribute CustomerId="abc" \
  --memo team=orders

# Blocking, prints result
temporal workflow execute \
  --workflow-id ping-42 --type Ping --task-queue util \
  --execution-timeout 1m
```

Common start flags: `--type` (req), `--task-queue` (req), `--workflow-id`, `--input`, `--input-file`, `--memo`, `--cron`, `--execution-timeout`, `--run-timeout`, `--task-timeout`, `--start-delay`, `--headers`, `--search-attribute`, `--id-conflict-policy`, `--id-reuse-policy`, `--fail-existing`, `--priority-key`, `--fairness-key`, `--fairness-weight`.

## Signal / Query / Update

```bash
# Single Signal
temporal workflow signal \
  --workflow-id order-42 \
  --name approve \
  --input '{"approver":"hs094"}'

# Bulk Signal via query (spawns a batch job — see CL 04)
temporal workflow signal \
  --query 'WorkflowType="OrderWorkflow" AND ExecutionStatus="Running"' \
  --name pause --reason "deploy freeze" --yes

# Query for state
temporal workflow query \
  --workflow-id order-42 --type currentStatus

# Update — synchronous RPC handler
temporal workflow update execute \
  --workflow-id order-42 \
  --name applyDiscount \
  --input '{"pct":15}'

temporal workflow update start \
  --workflow-id order-42 --name applyDiscount \
  --wait-for-stage accepted

# Signal-with-start: idempotent kickoff
temporal workflow signal-with-start \
  --signal-name newOrder \
  --signal-input '{"sku":"A1"}' \
  --workflow-id order-42 \
  --type OrderWorkflow --task-queue orders
```

## Inspecting

```bash
temporal workflow describe --workflow-id order-42
temporal workflow describe --workflow-id order-42 --reset-points true
temporal workflow show --workflow-id order-42 --output json
temporal workflow show --workflow-id order-42 --follow
temporal workflow result --workflow-id order-42

temporal workflow list --query 'ExecutionStatus="Running"'
temporal workflow list --archived --limit 50
temporal workflow count --query 'WorkflowType="OrderWorkflow"'

temporal workflow trace --workflow-id order-42 --depth 3 --fold canceled
temporal workflow stack --workflow-id order-42
```

## Ending Workflows

```bash
# Graceful — Workflow code sees CancellationException
temporal workflow cancel --workflow-id order-42 --reason "user requested"

# Forced — no cleanup
temporal workflow terminate --workflow-id order-42 --reason "stuck"

# Bulk terminate via query (batch job)
temporal workflow terminate \
  --query 'WorkflowType="OrderWorkflow" AND StartTime<"2025-01-01"' \
  --reason "old test data" --yes

# Reset to before the bad activity
temporal workflow reset \
  --workflow-id order-42 --event-id 27 \
  --reason "skip broken activity"

# Reset to last continue-as-new
temporal workflow reset \
  --workflow-id order-42 --type LastContinuedAsNew

# Delete history entirely (async)
temporal workflow delete --workflow-id order-42 --reason "GDPR"
```

## update-options

Mutate Workflow versioning at runtime:

```bash
temporal workflow update-options \
  --workflow-id order-42 \
  --versioning-override-behavior pinned \
  --versioning-override-deployment-name orders \
  --versioning-override-build-id v2.1.0
```

## Common patterns

```bash
# Smoke-test deploy: list any failed in last hour
temporal workflow list \
  --query 'ExecutionStatus="Failed" AND CloseTime>"2026-04-27T00:00:00Z"'

# Drain all running instances of a deprecated WF type
temporal workflow cancel \
  --query 'WorkflowType="OldFlow" AND ExecutionStatus="Running"' \
  --reason "deprecated" --yes

# Rescue a stuck workflow: stack + reset
temporal workflow stack --workflow-id order-42
temporal workflow reset --workflow-id order-42 --type LastWorkflowTask
```

⚠️ `terminate` is **non-graceful**: no cancellation handlers run, no compensations fire. Prefer `cancel`. Use `terminate` only when the Workflow is wedged or unresponsive.

⚠️ `delete` removes the history — irreversible. Only use post-`terminate`/`complete` and only when retention isn't sufficient (e.g., GDPR).

⚠️ Any `--query` form on `signal/cancel/terminate/delete/reset/update-options` spawns a **batch job** (see [[Temporal CL 04 - batch]]). Always dry-run the query with `workflow list` first; once a batch fires you can only `batch terminate` it, not undo it.

⚠️ `reset` with `--event-id N` rewinds history to event N — every event after N is replayed from worker code. Good fit for "skip the broken activity attempt", bad fit for "I changed my Workflow type signature".

⚠️ `--id-reuse-policy` and `--id-conflict-policy` are different things — reuse handles closed runs, conflict handles open runs. Mix-up causes spurious duplicate errors.

💡 **Takeaway:** start/execute to launch, signal/query/update to interact, describe/show/list to inspect, cancel/terminate/delete/reset to end — and `--query` everywhere for bulk ops via batch jobs.
