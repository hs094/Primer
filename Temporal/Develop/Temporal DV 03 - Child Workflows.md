# DV 03 — Child Workflows

🔑 **`workflow.execute_child_workflow(...)` runs another workflow as a sub-orchestration within the parent's history; use it to compose, fan out, or isolate concerns.**

## Two entry points

| Call | Returns | Use when |
|---|---|---|
| `workflow.execute_child_workflow(...)` | result (awaits completion) | you just need the result |
| `workflow.start_child_workflow(...)` | `ChildWorkflowHandle` | you want the run ID, want to signal it, or run it concurrently |

`execute_child_workflow` is sugar for `start_child_workflow` + `await handle`.

## Basic usage

```python
from dataclasses import dataclass
from temporalio import workflow

@dataclass
class ComposeGreetingInput:
    greeting: str
    name: str

@workflow.defn
class ComposeGreetingWorkflow:
    @workflow.run
    async def run(self, input: ComposeGreetingInput) -> str:
        return f"{input.greeting}, {input.name}!"

@workflow.defn
class GreetingWorkflow:
    @workflow.run
    async def run(self, name: str) -> str:
        return await workflow.execute_child_workflow(
            ComposeGreetingWorkflow.run,
            ComposeGreetingInput("Hello", name),
            id="hello-child-workflow-workflow-child-id",
        )
```

Both parent and child must be registered on a worker that polls the relevant [[Temporal CL 10 - task-queue]] (same queue by default; pass `task_queue=` to route elsewhere).

## Get a handle, run concurrently

```python
@workflow.defn
class GreetingWorkflow:
    @workflow.run
    async def run(self, name: str) -> str:
        handle = await workflow.start_child_workflow(
            ComposeGreetingWorkflow.run,
            ComposeGreetingInput("Hello", name),
            id="hello-child-workflow-workflow-child-id",
        )
        # ... do other work, signal the child, etc.
        return await handle
```

The handle exposes `.id`, `.first_execution_run_id`, `.signal(...)`, `.cancel()`. Background details: [[Temporal EN 09 - Child Workflows]].

## Parent Close Policy

Defines what happens to the child when the parent closes (Completed / Failed / Timed Out). Default: child is **terminated** with the parent.

```python
from temporalio.workflow import ParentClosePolicy

return await workflow.execute_child_workflow(
    ComposeGreetingWorkflow.run,
    ComposeGreetingInput("Hello", name),
    id="hello-child-workflow-workflow-child-id",
    parent_close_policy=ParentClosePolicy.ABANDON,
)
```

Options:

- `TERMINATE` (default) — child cancelled when parent closes.
- `ABANDON` — child runs independently after parent closes.
- `REQUEST_CANCEL` — graceful cancellation request to the child.

Use `ABANDON` for fire-and-forget side workflows that should outlive the parent.

## Child vs Activity — when to reach for it

Pick a child workflow when you need:

- A separate event history (parent's history doesn't bloat).
- Independent retry/timeout policy as a unit.
- Independent message passing surface (signals, queries, updates) — see [[Temporal DV 07 - Message Passing]].
- Cross-team or cross-service composition where the child is an independently versioned workflow.

Pick an Activity when the work is a single side effect — see [[Temporal DV 10 - Activity Basics]].

⚠️ **Child Workflow IDs collide globally.** A child's `id` lives in the same namespace as any other workflow ID — pick something deterministic but unique (e.g. `f"compose-{parent_id}-{idx}"`), or expect `WorkflowAlreadyStartedError` on replay.

⚠️ **Cancelling a child is not the same as cancelling its parent.** If you want graceful child shutdown when the parent closes, set `parent_close_policy=REQUEST_CANCEL` and have the child handle `CancelledError` — see [[Temporal DV 05 - Workflow Cancellation]].

💡 **Takeaway:** Reach for a child workflow when you need an independent unit of orchestration, retry policy, or message-passing surface — otherwise an activity is cheaper.
