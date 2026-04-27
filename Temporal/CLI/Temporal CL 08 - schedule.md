# CL 08 — schedule

🔑 **One-line: manage [[Temporal DV 08 - Schedules]] — cron/calendar/interval-driven Workflow Executions on the Service.**

Eight subcommands cover the full lifecycle. A Schedule wraps a Workflow start request with timing rules (calendar / interval / cron) plus overlap policy, jitter, catchup, and pause-on-failure semantics.

## Subcommands

| Subcommand | Purpose |
|---|---|
| `create` | Register a new Schedule |
| `update` | Replace a Schedule's spec (full overwrite) |
| `describe` | Show config + recent/upcoming runs |
| `list` | List all Schedules in namespace |
| `toggle` | Pause / unpause |
| `trigger` | Fire one execution now, off-schedule |
| `backfill` | Replay missed executions over a time window |
| `delete` | Remove the Schedule (running children survive) |

## Scheduling specs

At least one of these on `create`/`update`:

| Flag | Format |
|---|---|
| `--calendar` | JSON: `'{"dayOfWeek":"Fri","hour":"3","minute":"30"}'` |
| `--interval` | Duration: `"45m"` or offset `"6h/5h"` (every 6h, starting hour 5) |
| `--cron` | Unix cron: `"30 12 * * Fri"` or `@daily`, `@every 1h` |

## create

```bash
temporal schedule create \
  --schedule-id nightly-rollup \
  --task-queue analytics \
  --type RollupWorkflow \
  --calendar '{"hour":"2","minute":"15"}' \
  --time-zone America/Los_Angeles \
  --overlap-policy Skip \
  --pause-on-failure \
  --input '{"region":"us-west"}' \
  --memo team=data
```

Key optional flags:

| Flag | Purpose |
|---|---|
| `--workflow-id, -w` | Workflow ID (auto if omitted) |
| `--start-time`, `--end-time` | Activation window |
| `--time-zone` | TZ database name (calendar specs) |
| `--paused` | Create paused |
| `--pause-on-failure` | Auto-pause after Workflow failure |
| `--jitter` | Randomise start within duration |
| `--catchup-window` | Max catch-up after Service unavailability |
| `--overlap-policy` | `Skip`, `BufferOne`, `BufferAll`, `CancelOther`, `TerminateOther`, `AllowAll` |
| `--remaining-actions` | Cap total runs (`0` = unlimited) |
| `--execution-timeout`, `--run-timeout`, `--task-timeout` | Per-run timeouts |
| `--input`, `--input-file`, `--input-base64`, `--input-meta` | Workflow inputs |
| `--memo`, `--schedule-memo`, `--search-attribute`, `--schedule-search-attribute` | Metadata |
| `--priority-key` (1–5), `--fairness-key`, `--fairness-weight` | Scheduling priority |

## update / describe / list

```bash
# update is a full replacement — re-pass everything you want preserved
temporal schedule describe --schedule-id nightly-rollup
temporal schedule update --schedule-id nightly-rollup \
  --task-queue analytics --type RollupWorkflow \
  --calendar '{"hour":"3","minute":"15"}' --overlap-policy Skip

temporal schedule list --query 'WorkflowType="RollupWorkflow"'
temporal schedule list --long
```

## toggle / trigger

```bash
temporal schedule toggle --schedule-id nightly-rollup --pause   --reason "deploy freeze"
temporal schedule toggle --schedule-id nightly-rollup --unpause --reason "freeze lifted"

temporal schedule trigger --schedule-id nightly-rollup \
  --overlap-policy BufferAll
```

## backfill

Replay every action that *would* have fired in `[start, end]`:

```bash
temporal schedule backfill \
  --schedule-id nightly-rollup \
  --start-time 2026-04-01T00:00:00Z \
  --end-time   2026-04-07T00:00:00Z \
  --overlap-policy BufferAll
```

## delete

```bash
temporal schedule delete --schedule-id nightly-rollup
# Workflows already started by this schedule keep running.
```

## Common patterns

```bash
# Every weekday 09:00 PT, skip if last run still going
temporal schedule create --schedule-id morning-sync \
  --task-queue ops --type SyncWorkflow \
  --calendar '{"dayOfWeek":"Mon-Fri","hour":"9","minute":"0"}' \
  --time-zone America/Los_Angeles \
  --overlap-policy Skip

# Hourly with 5-minute jitter to avoid thundering herd
temporal schedule create --schedule-id heartbeat \
  --task-queue util --type Heartbeat \
  --interval 1h --jitter 5m
```

⚠️ `update` is a **full replacement**, not a patch. Run `describe` first and re-pass every flag you want to preserve, otherwise you'll wipe overlap policy / timeouts.

⚠️ Schedule **memo** and **search attributes** cannot be changed via `update` — set them at `create` time. `--memo` updates Workflow memo, not schedule memo.

⚠️ `delete` does **not** stop in-flight Workflow Executions started by the schedule — it only stops further ones. Cancel/terminate the children separately if needed.

⚠️ Calendar specs are JSON, not cron. Mixing: `--cron` is plain string, `--calendar` is JSON object — easy to swap by accident.

💡 **Takeaway:** `create/update/describe/list` for spec management, `toggle/trigger/backfill` for ops, `delete` to stop scheduling — children persist on delete.
