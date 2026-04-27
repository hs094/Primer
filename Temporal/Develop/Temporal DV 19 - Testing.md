# DV 19 — Testing

🔑 Three test layers: **unit** (pure functions), **integration** (mocked Activities + real Workflow engine via `WorkflowEnvironment`), **end-to-end** (real Server, real Workers). Most of your tests should be integration — they catch determinism bugs without the cost of full E2E.

## Tooling

`pytest` is the recommended runner — fixtures, parametrize, async support all play well. Install:

```bash
uv add --dev pytest pytest-asyncio
```

## Testing Activities in isolation

Use `ActivityEnvironment` — no Worker, no Service, just the Activity:

```python
from temporalio import activity
from temporalio.testing import ActivityEnvironment

@activity.defn
async def activity_with_heartbeats(param: str):
    activity.heartbeat(f"param: {param}")
    activity.heartbeat("second heartbeat")

env = ActivityEnvironment()
heartbeats = []
env.on_heartbeat = lambda *args: heartbeats.append(args[0])
await env.run(activity_with_heartbeats, "test")
assert heartbeats == ["param: test", "second heartbeat"]
```

`on_heartbeat` is a callable invoked every time the Activity calls `activity.heartbeat()` — perfect for asserting progress signaling. See [[Temporal DV 13 - Activity Timeouts]].

## Testing Workflows with mocked Activities

The standard pattern: spin up a `WorkflowEnvironment`, register the real Workflow with **mock Activities**, and execute.

```python
import uuid
from temporalio import activity
from temporalio.client import Client
from temporalio.worker import Worker
from hello.hello_activity import (
    ComposeGreetingInput, GreetingWorkflow, compose_greeting,
)

@activity.defn(name="compose_greeting")
async def compose_greeting_mocked(input: ComposeGreetingInput) -> str:
    return f"{input.greeting}, {input.name} from mocked activity!"

async def test_mock_activity(client: Client):
    task_queue_name = str(uuid.uuid4())
    async with Worker(
        client,
        task_queue=task_queue_name,
        workflows=[GreetingWorkflow],
        activities=[compose_greeting_mocked],
    ):
        assert "Hello, World from mocked activity!" == await client.execute_workflow(
            GreetingWorkflow.run,
            "World",
            id=str(uuid.uuid4()),
            task_queue=task_queue_name,
        )
```

⚠️ The mock **must have the same name** as the real Activity (`@activity.defn(name="compose_greeting")`). The Workflow looks up Activities by name, not by Python identity.

## WorkflowEnvironment flavors

```python
from temporalio.testing import WorkflowEnvironment
```

| Method | Use |
|---|---|
| `WorkflowEnvironment.start_time_skipping()` | Fast in-process Server with **time skipping** — best for most tests |
| `WorkflowEnvironment.start_local()` | Real local Server; no time skipping; closer to prod |
| `WorkflowEnvironment.from_client(client)` | Wrap an existing Client (e.g. dev server) |

## Time skipping

Workflows that `await workflow.sleep(timedelta(days=30))` would block tests. With time skipping, the test environment fast-forwards virtual time when no work is pending.

```python
async def test_manual_time_skipping():
    async with await WorkflowEnvironment.start_time_skipping() as env:
        await env.sleep(3)  # advances virtual time by 3 seconds
```

Combined with `start_workflow` + delays, you can verify a 30-day timer in milliseconds:

```python
async with await WorkflowEnvironment.start_time_skipping() as env:
    async with Worker(env.client, task_queue="q", workflows=[W]):
        handle = await env.client.start_workflow(W.run, id="x", task_queue="q")
        await env.sleep(timedelta(days=30))
        assert (await handle.result()) == "expired"
```

## Replay tests

Replay tests guard against determinism breaks when you change a Workflow Definition. Pull histories from prod and replay them against the new code:

```python
from temporalio.worker import Replayer
from temporalio.client import WorkflowHistory

# Many histories at once
workflows = client.list_workflows(
    "TaskQueue=foo and StartTime > '2022-01-01T12:00:00'"
)
histories = workflows.map_histories()
replayer = Replayer(workflows=[MyWorkflowA, MyWorkflowB, MyWorkflowC])
await replayer.replay_workflows(histories)

# Single history from JSON
replayer = Replayer(workflows=[YourWorkflow])
await replayer.replay_workflow(WorkflowHistory.from_json(history_json_str))
```

The recommended practice: **fail CI if any history fails to replay**. This catches non-deterministic refactors before they reach prod and break in-flight Workflows. See [[Temporal BP 03 - Pre-Production Testing]].

## Assertions in Workflow code

`assert` works inside Workflow definitions — useful for invariant checks during development:

```python
@workflow.defn
class W:
    @workflow.run
    async def run(self, n: int) -> int:
        assert n >= 0, "n must be non-negative"
        ...
```

⚠️ Failed asserts raise `AssertionError`, which becomes a Workflow Task failure (retried) — not a Workflow failure. If you want the Workflow to fail, raise `ApplicationError(non_retryable=True)` instead. See [[Temporal EN 05 - Detecting Application Failures]].

## pytest fixture pattern

A reusable fixture that gives every test a fresh Client:

```python
import pytest
from temporalio.testing import WorkflowEnvironment

@pytest.fixture
async def client():
    async with await WorkflowEnvironment.start_time_skipping() as env:
        yield env.client
```

Then:

```python
async def test_my_workflow(client):
    async with Worker(client, task_queue="q", workflows=[W], activities=[a]):
        result = await client.execute_workflow(W.run, "hi", id="x", task_queue="q")
        assert result == "hello hi"
```

## Cross-references

- [[Temporal DV 18 - Observability]] — metrics/logs to inspect during tests
- [[Temporal DV 22 - Debugging]] — replayer is also a debug tool
- [[Temporal BP 03 - Pre-Production Testing]] — strategy
- [[Temporal DV 20 - Sandbox and Sync vs Async]] — sandbox can interfere with mocks

💡 **Takeaway:** Use `ActivityEnvironment` for Activity unit tests, `WorkflowEnvironment.start_time_skipping()` + mock Activities for Workflow integration tests, and `Replayer` against real histories to catch determinism regressions in CI.
