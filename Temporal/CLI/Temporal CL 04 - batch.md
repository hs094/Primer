# CL 04 — batch

🔑 **One-line: inspect and terminate the batch jobs that bulk-Workflow commands (`workflow signal --query`, `workflow terminate --query`, etc.) create on the server.**

Whenever you run a Workflow command with `--query` against many executions (signal, terminate, cancel, reset, delete, update-options), the Service spawns a **batch job** to execute it asynchronously. `temporal batch` lets you watch and abort those jobs.

## Subcommands

| Subcommand | Purpose |
|---|---|
| `list` | List batch jobs (current namespace) |
| `describe` | Show progress of one batch job |
| `terminate` | Abort an in-flight batch job (reason required) |

There is **no** `batch start` or `batch create` — batch jobs are created implicitly by other CLI/SDK calls.

## Examples

```bash
# See running and recent batch jobs
temporal batch list --namespace prod

# Cap the output
temporal batch list --limit 20

# Drill into one job
temporal batch describe --job-id <uuid>

# Stop a runaway batch (e.g. signal flooding millions of WFs)
temporal batch terminate \
  --job-id <uuid> \
  --reason "wrong query — was selecting all WFs"
```

## What creates batch jobs

Any of these `--query`-driven commands spawns one:

| Triggering command | What the batch does |
|---|---|
| `workflow signal --query ...` | Signals every matching Workflow |
| `workflow cancel --query ...` | Cancels every matching Workflow |
| `workflow terminate --query ...` | Force-terminates every match |
| `workflow delete --query ...` | Deletes histories of matches |
| `workflow reset --query ...` | Resets each match to a chosen point |
| `workflow update-options --query ...` | Mutates options on each match |
| `activity reset --query ...` | Resets activities on matching Workflows |
| `activity unpause --query ...` | Unpauses activities on matching WFs |

The triggering command **prints the batch job ID** on success — capture it for `describe`/`terminate`.

## Common patterns

```bash
# Workflow command that triggers a batch
temporal workflow signal \
  --query 'WorkflowType="OrderWorkflow" AND ExecutionStatus="Running"' \
  --name pause \
  --reason "deploy freeze"
# -> prints a batch job ID

# Watch progress until done
temporal batch describe --job-id <id> --output json | jq '.state'

# Discover and terminate a runaway batch by listing recent ones
temporal batch list --limit 5 --output json \
  | jq -r '.[] | select(.state=="Running") | .jobId'
```

⚠️ `batch terminate` requires `--reason` — the Service stores it as metadata, surfaced in [[Temporal CL 12 - workflow]] events on each affected Workflow. Don't omit it; future-you will thank present-you.

⚠️ Batch jobs are **per-namespace**. Switch `--namespace` to see jobs in another namespace; there is no cross-namespace listing.

⚠️ Once a batch job is terminated, the Workflows already processed remain affected — `terminate` only stops further actions, it does **not** roll back what's already been signalled/cancelled.

💡 **Takeaway:** any `--query`-based bulk command spawns a batch; `temporal batch list/describe/terminate` is the rescue path when the query was wrong.
