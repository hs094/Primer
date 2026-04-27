# DV 22 — Debugging

🔑 Debugging Temporal is mostly **observation, not breakpoints**. Use the Web UI for the event history, the Replayer for "did my new code break old runs", and `workflow.logger` for the actual narrative. Set a debugger only inside Activities — Workflow code re-executes on replay.

## Two phases, two toolkits

| Phase | Tools |
|---|---|
| **Development** | Web UI + `temporal` CLI + stdlib debugger inside Activities + `workflow.logger` inside Workflows |
| **Production** | Web UI + Replayer against captured histories + traces + metrics + logs |

## The Web UI

`http://localhost:8233` (dev server) or your Cloud namespace URL. For each Workflow Execution you can see:

- **Event History** — the full ordered list of events. This is the source of truth.
- **Pending Activities** — what's queued/running.
- **Stack Trace** (via the built-in `__stack_trace` query) — where the Workflow is currently `await`-ing.
- **Input / Result** — payloads (decrypted only if you've wired a codec server, see [[Temporal DV 21 - Data Handling]]).

Open this **first** for any "what happened" question.

## CLI inspection

Mirror everything in the UI from the terminal — useful in CI and when the UI is unavailable.

```bash
temporal workflow describe -w <ID>     # status + summary
temporal workflow show -w <ID>          # full event history
temporal workflow stack -w <ID>         # current stack trace via query
temporal workflow list --query 'WorkflowType="Foo" and ExecutionStatus="Failed"'
```

The `stack` command is gold for hung Workflows — it tells you exactly which `await` is parked.

## Logging

Use `workflow.logger` inside Workflows so messages are **replay-aware** (suppressed on replay so logs match real timeline):

```python
from temporalio import workflow

@workflow.defn
class W:
    @workflow.run
    async def run(self, x: int):
        workflow.logger.info(f"starting with {x}")
        ...
```

Inside Activities, plain stdlib logging works — Activities don't replay:

```python
import logging
logger = logging.getLogger(__name__)

@activity.defn
async def fetch(url: str):
    logger.info(f"fetching {url}")
```

⚠️ `print()` inside a Workflow runs on every replay and floods stdout. Use `workflow.logger` exclusively.

## Setting breakpoints

| Code path | Breakpoint? |
|---|---|
| Activity function | Yes — Activities run once, no replay |
| Workflow function | Risky — Workflow code re-executes on replay; you'll hit the same breakpoint multiple times |
| Worker setup | Yes |

For Workflow logic, prefer **logging + Replayer** to step through deterministically.

## The Replayer (the time machine)

The single most powerful debugging tool: take a real history, replay it against your local code, watch it execute step-by-step.

```python
from temporalio.worker import Replayer
from temporalio.client import WorkflowHistory

replayer = Replayer(workflows=[YourWorkflow])
await replayer.replay_workflow(WorkflowHistory.from_json(history_json_str))
```

Steps to debug a prod failure:

1. Pull the history: `temporal workflow show -w <ID> --output json > history.json`
2. Load it locally with `WorkflowHistory.from_json(open("history.json").read())`.
3. Set breakpoints in your Workflow code — replay walks them deterministically.
4. Iterate on a fix; replay again to confirm.

Bulk replay against many histories catches determinism breaks before you ship — see [[Temporal DV 19 - Testing]] and [[Temporal BP 03 - Pre-Production Testing]].

## Tracing

OpenTelemetry traces, configured in [[Temporal DV 18 - Observability]], give you the full call graph: Client → Workflow → Activity → Child / Nexus. Spans tag every step with workflow ID, run ID, attempt count — so a single trace ID lets you reconstruct multi-Workflow flows.

## Metrics

Watch in [[Temporal RF 01 - SDK Metrics]] when something feels slow:

- `temporal_workflow_task_schedule_to_start_latency` ↑ → Worker pool too small
- `temporal_workflow_task_replay_latency` ↑ → expensive replay (large history?)
- `temporal_activity_execution_failed` ↑ → upstream broken
- `temporal_sticky_cache_size` near limit → cache thrashing, more replays

## Common pitfalls

| Symptom | Likely cause |
|---|---|
| `NonDeterminismError` on replay | Code change broke history compatibility — restore old logic via versioning ([[Temporal PD 02 - Worker Deployments]]) or run only with `start_workflow` versioning gates |
| Workflow stuck "running" forever | Awaiting a Signal that never fires; check `temporal workflow stack -w <ID>` |
| Activity retried infinitely | Non-retryable error not flagged — raise `ApplicationError(non_retryable=True)` |
| Worker frozen, no progress | Blocking call in async Activity froze event loop — see [[Temporal DV 20 - Sandbox and Sync vs Async]] |
| Web UI shows ciphertext | No codec server hooked up — see [[Temporal DV 21 - Data Handling]] |
| `Could not be encoded for sandbox` | Module touches non-deterministic stdlib in workflow scope — pass through or move to Activity |

## A debugging checklist

When something looks wrong:

1. **Find the Workflow** in the UI — get the run ID.
2. **Read the History** — what was the last successful event? What event failed?
3. **`workflow.logger` output** — line up logs with history events.
4. **Stack trace query** — where is it parked right now?
5. **Reproduce locally with Replayer** — set breakpoints, step.
6. **Check metrics + traces** for the surrounding window.

## Cross-references

- [[Temporal DV 18 - Observability]] — metrics, traces, logs, queries
- [[Temporal DV 19 - Testing]] — Replayer in CI
- [[Temporal PD 02 - Worker Deployments]] — fixing determinism breaks safely
- [[Temporal CL 02 - Setup CLI]] — `temporal` commands
- [[Temporal BP 03 - Pre-Production Testing]] — Replay-against-prod-histories strategy

💡 **Takeaway:** Treat the Event History as ground truth. Use the UI/CLI to read it, `workflow.logger` to narrate it, the Replayer to walk through it locally with breakpoints, and traces/metrics to find the surrounding context. Never set breakpoints in Workflow code without using the Replayer.
