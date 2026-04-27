# GS 02 — Understanding Temporal

🔑 **Workflows + Activities run on Workers; the Temporal Service persists every step so anything can resume from history.**

## The mental model
Four moving parts. Once you have them, the rest of Temporal is mostly configuration.

| Piece | Role |
|---|---|
| Workflow | Your business logic as code. Deterministic. |
| Activity | A unit of work that touches the outside world (HTTP, DB, email). Non-deterministic; retried by the platform. |
| Worker | A process you run that polls Task Queues and executes Workflow / Activity code. |
| Temporal Service | The orchestrator + database. Records every event, dispatches tasks, replays history on failure. |

See [[Temporal EN 01 - Temporal Platform]] for the architecture diagram.

## Durable Execution, concretely
The Temporal Service keeps an Event History per Workflow Execution: every command issued, every Activity scheduled, every result returned, every timer fired. If a Worker dies mid-execution, another Worker picks up the same Workflow, replays the history, and resumes at the next unrecorded step.

The implication for your code: Workflows must be deterministic. Same input + same history = same decisions. Random IDs, wall-clock time, network calls — all forbidden inside a Workflow body. They go in Activities.

```python
# Workflow: deterministic, just orchestrates
@workflow.defn
class SayHelloWorkflow:
    @workflow.run
    async def run(self, name: str) -> str:
        return await workflow.execute_activity(
            greet,
            name,
            schedule_to_close_timeout=timedelta(seconds=10),
        )
```

```python
# Activity: anything goes — HTTP, DB, files, time
@activity.defn
async def greet(name: str) -> str:
    return f"Hello {name}"
```

More on the model: [[Temporal DV 02 - Workflow Basics]] and [[Temporal DV 10 - Activity Basics]].

## How Workers fit in
A Worker is a long-lived process you operate. It connects to the Temporal Service, polls a named Task Queue, and runs the Workflow / Activity code registered with it.

```python
worker = Worker(
    client,
    task_queue="my-task-queue",
    workflows=[SayHelloWorkflow],
    activities=[greet],
)
await worker.run()
```

Scale by running more Workers. Isolate by giving them different Task Queues. Code runs in *your* infrastructure with *your* secrets — the Temporal Service never sees Activity payloads beyond what you let it. See [[Temporal DV 15 - Worker Processes]].

## How the Temporal Service fits in
The Service is the only stateful component. It:

- Receives commands ("start workflow", "signal", "query")
- Persists Event History to a backing DB (Cassandra / Postgres / MySQL)
- Dispatches Workflow Tasks and Activity Tasks to Workers via Task Queues
- Enforces timeouts and retry policies
- Serves the Web UI and CLI

Your code never embeds the Service. It connects via the SDK client. See [[Temporal EN 11 - Temporal Service]].

## Replay = recovery
When a Worker reloads a Workflow:

1. It fetches the Event History from the Service.
2. It re-executes the Workflow function from the top.
3. For each step already in history (e.g. `execute_activity` that completed), the SDK returns the recorded result instead of running it again.
4. Execution continues at the first step *not* in history.

This is why determinism matters: replay must produce identical commands.

## Visibility
- **Web UI** at `http://localhost:8233` (dev server) — list Executions, view Event History, inspect stack traces.
- **CLI** for scripting: `temporal workflow list`, `temporal workflow show -w <id>`. See [[Temporal CL 01 - CLI Overview]].

⚠️ Putting `time.sleep()`, `random.random()`, `requests.get()`, or `datetime.now()` directly in a Workflow function will break replay non-deterministically. Use `workflow.sleep`, `workflow.random()`, `workflow.now()`, or push the call into an Activity.

💡 **Takeaway:** Workflow = deterministic orchestration; Activity = side effects; Worker = your process; Service = durable history + dispatch.
