# CL 03 — activity

🔑 **One-line: control [[Temporal EN 04 - Activities]] — both Standalone Activities and Activities running inside Workflows.**

Two flavours of subcommands here: lifecycle ops on **Standalone Activities** (`start`, `execute`, `result`, `terminate`, `cancel`) and Workflow-side controls for activities owned by a Workflow Execution (`pause`, `unpause`, `reset`, `complete`, `fail`, `update-options`).

## Subcommands

| Subcommand | Purpose |
|---|---|
| `start` | Start a new Standalone Activity, return Activity ID + Run ID |
| `execute` | Start a Standalone Activity and block until it completes |
| `result` | Wait for and print the result of a Standalone Activity |
| `describe` | Show details for a Standalone Activity |
| `list` | List Standalone Activities (supports `--query`) |
| `count` | Count Standalone Activities matching a `--query` |
| `cancel` | Request cancellation of a Standalone Activity |
| `terminate` | Force-terminate a Standalone Activity |
| `complete` | Externally mark a Workflow's Activity as succeeded with `--result` JSON |
| `fail` | Externally mark a Workflow's Activity as failed |
| `pause` | Pause an Activity inside a running Workflow |
| `unpause` | Resume a previously-paused Activity |
| `reset` | Restart an Activity as if first scheduled (clears attempts/timeouts) |
| `update-options` | Mutate timeouts, retry policy, task queue at runtime |

## Standalone Activity flow

```bash
# Start, then later collect the result
temporal activity start \
  --activity-id charge-abc \
  --type ChargeCustomer \
  --task-queue billing \
  --start-to-close-timeout 30s \
  --input '{"amount":4200}'

temporal activity result --activity-id charge-abc

# One-shot blocking
temporal activity execute \
  --activity-id ping \
  --type Ping \
  --task-queue util \
  --start-to-close-timeout 10s
```

## Workflow-attached operations

Most flags accept either a single target (`--workflow-id` + `--activity-id` or `--activity-type`) **or** a bulk selector (`--query`, `--match-all`):

```bash
# Async-completion: external worker reports success
temporal activity complete \
  --workflow-id order-42 \
  --activity-id ship \
  --result '{"tracking":"1Z..."}'

temporal activity fail \
  --workflow-id order-42 \
  --activity-id ship \
  --reason CarrierDown \
  --detail '{"code":503}'

# Pause every retrying activity in a Workflow
temporal activity pause \
  --workflow-id order-42 \
  --match-all

# Reset all activities of a given type, namespace-wide
temporal activity reset \
  --activity-type ChargeCustomer \
  --query 'WorkflowType="OrderWorkflow"' \
  --reset-attempts --reset-heartbeats
```

## update-options (mutate live config)

```bash
temporal activity update-options \
  --workflow-id order-42 \
  --activity-id ship \
  --start-to-close-timeout 5m \
  --heartbeat-timeout 30s
```

## Common patterns

```bash
# List all activities currently failing in prod namespace
temporal activity list --query 'ActivityStatus="Failed"'

# Bulk-unpause everything paused yesterday after a deploy
temporal activity unpause --query 'ActivityStatus="Paused"'

# Cancel a long-running standalone activity safely
temporal activity cancel --activity-id long-batch --reason "manual abort"
```

⚠️ `terminate` is brutal — Activity code cannot observe or react to it (no cleanup, no heartbeat, no cancellation handler runs). Prefer `cancel` whenever the Activity is well-behaved.

⚠️ `reset` issues a Canceled failure to any activity that's currently heartbeating. The next attempt starts clean — but the in-flight attempt sees a cancellation, not a fresh start.

⚠️ `complete`/`fail` are for **async-completion** activities (where the worker calls `ActivityCompletionClient` or returns `ActivityNotCompleted`). Calling them on a regular activity that's actively running will surprise you.

💡 **Takeaway:** `activity start/execute` for [[Temporal DV 02 - Workflow Basics|standalone]] runs; `pause`/`reset`/`update-options` for surgical control of activities inside live Workflows — both bulkable with `--query`.
