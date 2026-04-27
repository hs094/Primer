# DV 04 — Continue As New

🔑 **`workflow.continue_as_new(...)` atomically closes the current run and starts a fresh one with the same Workflow ID — used to keep long-running workflows under the event-history limit.**

## Why it exists

Every workflow signal, activity, timer, etc. appends events to [[Temporal EN 07 - Event History]]. Histories have hard limits (50k events, ~50MB) and soft costs (replay time scales with history size). Continue-As-New gives you a checkpoint: keep the Workflow ID stable, reset the Run ID and history, carry forward only the state you need.

## Pattern: pass forward state via dataclass

```python
from dataclasses import dataclass
from typing import Optional
from temporalio import workflow

@dataclass
class ClusterManagerInput:
    state: Optional[ClusterManagerState] = None
    test_continue_as_new: bool = False

@workflow.defn
class ClusterManagerWorkflow:
    @workflow.run
    async def run(self, input: ClusterManagerInput) -> ClusterManagerResult:
        self.state = input.state or ClusterManagerState()
        # ... long-running loop ...
        if self.should_continue_as_new(input):
            workflow.continue_as_new(
                ClusterManagerInput(
                    state=self.state,
                    test_continue_as_new=input.test_continue_as_new,
                )
            )
```

`continue_as_new(...)` raises a `ContinueAsNewError` internally — the workflow stops at that point. Pass the same workflow function or a different one (rare) as the first arg if you want, but the dataclass-as-input pattern is the common shape.

## When to continue

Ask the SDK:

```python
def should_continue_as_new(self, input: ClusterManagerInput) -> bool:
    if workflow.info().is_continue_as_new_suggested():
        return True
    # Test hook: force CAN at smaller history sizes during tests
    if (
        input.test_continue_as_new
        and workflow.info().get_current_history_length() > 100
    ):
        return True
    return False
```

`is_continue_as_new_suggested()` flips true when the Server thinks history is getting heavy — the recommended default trigger. Check it at natural boundaries (after a poll loop iteration, between batches), not mid-step.

## Continuing without input changes

```python
workflow.continue_as_new()  # same args as the original run
```

If you don't pass new args, the run restarts with the original input.

## Test hook for fast iteration

The example above wires a `test_continue_as_new` flag through the input so you can force a CAN after only ~100 events in a [[Temporal DV 19 - Testing]] run rather than waiting for the suggestion. Production code uses `is_continue_as_new_suggested()` only.

⚠️ **Don't call `continue_as_new` from a signal/update handler.** Continue-As-New must originate from the `@workflow.run` method. The recommended pattern: handlers set state, the run loop checks `workflow.all_handlers_finished()` or a similar wait condition, then calls CAN. See [[Temporal DV 07 - Message Passing]].

⚠️ **Pending work is dropped.** Activities, timers, and child workflows that haven't completed when you call `continue_as_new` are cancelled. Drain or persist them into the new run's input first.

⚠️ **State must be serialisable.** Whatever you pack into the new input goes through the payload converter — the same constraints as workflow args apply.

## Operator's view

In the UI you'll see a chain of executions sharing one Workflow ID, each with its own Run ID, linked by `WorkflowExecutionContinuedAsNew` events. Clients can still target the Workflow ID — they hit the latest run. See [[Temporal CL 12 - workflow]].

💡 **Takeaway:** Use `workflow.info().is_continue_as_new_suggested()` to gate `workflow.continue_as_new(...)` from inside `@workflow.run`, never from a handler. Pass forward only the minimal state you need.
