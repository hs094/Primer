# DV 07 — Message Passing

🔑 **Three handler kinds: Signals (async, no return), Queries (sync, read-only), Updates (sync or async, validated, returnable). Together they let clients drive a running workflow.**

Background: [[Temporal EN 08 - Workflow Message Passing]].

## Signals — fire-and-forget mutation

```python
from dataclasses import dataclass
from temporalio import workflow

@dataclass
class ApproveInput:
    name: str

@workflow.defn
class GreetingWorkflow:
    def __init__(self) -> None:
        self.approved_for_release = False
        self.approver_name: str | None = None

    @workflow.signal
    def approve(self, input: ApproveInput) -> None:
        self.approved_for_release = True
        self.approver_name = input.name
```

Signals "cannot return a value. The response is sent immediately from the server, without waiting for the Workflow to process the Signal." Signal handlers may be sync `def` or `async def`.

Send from a client:

```python
await workflow_handle.signal(GreetingWorkflow.approve, ApproveInput(name="me"))
```

## Queries — synchronous read

```python
@dataclass
class GetLanguagesInput:
    include_unsupported: bool

@workflow.defn
class GreetingWorkflow:
    def __init__(self) -> None:
        self.greetings = {
            Language.CHINESE: "你好，世界",
            Language.ENGLISH: "Hello, world",
        }

    @workflow.query
    def get_languages(self, input: GetLanguagesInput) -> list[Language]:
        if input.include_unsupported:
            return list(Language)
        else:
            return list(self.greetings)
```

> A Query handler uses `def`, not `async def`. You can't perform async operations like executing an Activity in a Query handler.

```python
supported = await workflow_handle.query(
    GreetingWorkflow.get_languages,
    GetLanguagesInput(include_unsupported=False),
)
```

## Updates — validated, returnable

```python
@workflow.defn
class GreetingWorkflow:
    @workflow.update
    def set_language(self, language: Language) -> Language:
        previous_language, self.language = self.language, language
        return previous_language

    @set_language.validator
    def validate_language(self, language: Language) -> None:
        if language not in self.greetings:
            raise ValueError(f"{language.name} is not supported")
```

The validator runs synchronously *before* the update is admitted to history — raise to reject. The handler runs after admission and may be sync or async.

Client side, three flavours:

```python
# Wait for completion + return value
previous = await workflow_handle.execute_update(
    GreetingWorkflow.set_language, Language.CHINESE,
)

# Get a handle, await separately
update_handle = await workflow_handle.start_update(
    HelloWorldWorkflow.set_greeting,
    HelloWorldInput("World"),
    wait_for_stage=client.WorkflowUpdateStage.ACCEPTED,
)
update_result = await update_handle.result()
```

## Async handlers can run activities

```python
import asyncio
from temporalio.exceptions import ApplicationError

@workflow.defn
class GreetingWorkflow:
    def __init__(self) -> None:
        self.lock = asyncio.Lock()

    @workflow.update
    async def set_language(self, language: Language) -> Language:
        if language not in self.greetings:
            async with self.lock:
                greeting = await workflow.execute_activity(
                    call_greeting_service,
                    language,
                    start_to_close_timeout=timedelta(seconds=10),
                )
                if greeting is None:
                    raise ApplicationError(
                        f"Greeting service does not support {language.name}"
                    )
                self.greetings[language] = greeting
        previous_language, self.language = self.language, language
        return previous_language
```

Use `asyncio.Lock` to serialise concurrent handlers that touch shared state — they run as cooperative coroutines on the workflow event loop, so awaits interleave.

## Wait conditions — sync the run loop with handlers

```python
@workflow.defn
class GreetingWorkflow:
    def __init__(self) -> None:
        self.approved_for_release = False

    @workflow.signal
    def approve(self, input: ApproveInput) -> None:
        self.approved_for_release = True

    @workflow.run
    async def run(self) -> str:
        await workflow.wait_condition(lambda: self.approved_for_release)
        return self.greetings[self.language]
```

Inside a handler:

```python
@workflow.update
async def my_update(self, update_input: UpdateInput) -> str:
    await workflow.wait_condition(
        lambda: self.ready_for_update_to_execute(update_input)
    )
    ...
```

Drain handlers before returning from `run`:

```python
@workflow.defn
class MyWorkflow:
    @workflow.run
    async def run(self) -> str:
        await workflow.wait_condition(workflow.all_handlers_finished)
        return "workflow-result"
```

## `@workflow.init` — share state with handlers from t=0

```python
@dataclass
class MyWorkflowInput:
    name: str

@workflow.defn
class WorkflowRunSeesWorkflowInitWorkflow:
    @workflow.init
    def __init__(self, workflow_input: MyWorkflowInput) -> None:
        self.name_with_title = f"Sir {workflow_input.name}"
        self.title_has_been_checked = False

    @workflow.run
    async def get_greeting(self, workflow_input: MyWorkflowInput) -> str:
        await workflow.wait_condition(lambda: self.title_has_been_checked)
        return f"Hello, {self.name_with_title}"

    @workflow.update
    async def check_title_validity(self) -> bool:
        is_valid = await workflow.execute_activity(
            check_title_validity,
            self.name_with_title,
            schedule_to_close_timeout=timedelta(seconds=10),
        )
        self.title_has_been_checked = True
        return is_valid
```

`@workflow.init` runs `__init__` with the workflow input — without it, handlers that arrive before `run` starts would see uninitialised state.

## Signal-With-Start

Atomic "start workflow if not running, then signal":

```python
await client.start_workflow(
    GreetingWorkflow.run,
    id="your-signal-with-start-workflow",
    task_queue="signal-tq",
    start_signal="submit_greeting",
    start_signal_args=["User Signal with Start"],
)
```

## Update-With-Start

```python
from temporalio import common
from temporalio.client import WithStartWorkflowOperation, WorkflowUpdateFailedError

start_op = WithStartWorkflowOperation(
    ShoppingCartWorkflow.run,
    id=cart_id,
    id_conflict_policy=common.WorkflowIDConflictPolicy.USE_EXISTING,
    task_queue="my-task-queue",
)
try:
    price = Decimal(
        await temporal_client.execute_update_with_start_workflow(
            ShoppingCartWorkflow.add_item,
            ShoppingCartItem(sku=item_id, quantity=quantity),
            start_workflow_operation=start_op,
        )
    )
except WorkflowUpdateFailedError:
    price = None
workflow_handle = await start_op.workflow_handle()
```

## Workflow-to-workflow signals

```python
@workflow.defn
class WorkflowB:
    @workflow.run
    async def run(self) -> None:
        handle = workflow.get_external_workflow_handle_for(WorkflowA.run, "workflow-a")
        await handle.signal(WorkflowA.your_signal, "signal argument")
```

## Dynamic handlers

```python
from typing import Sequence
from temporalio.common import RawValue

@workflow.defn
class MyWorkflow:
    @workflow.signal(dynamic=True)
    async def dynamic_signal(self, name: str, args: Sequence[RawValue]) -> None:
        ...

@workflow.defn(dynamic=True)
class DynamicWorkflow:
    @workflow.run
    async def run(self, args: Sequence[RawValue]) -> str:
        name = workflow.payload_converter().from_payload(args[0].payload, str)
        ...

@activity.defn(dynamic=True)
async def dynamic_greeting(args: Sequence[RawValue]) -> str:
    arg1 = activity.payload_converter().from_payload(args[0].payload, YourDataClass)
    return f"{arg1.greeting}, {arg1.name}!"
```

## Errors clients see

| Exception | Meaning |
|---|---|
| `RPCError(UNAVAILABLE)` | Server unreachable |
| `RPCError(NOT_FOUND)` | Workflow ID doesn't exist |
| `WorkflowUpdateFailedError` | Validator rejected, or handler raised |
| `WorkflowUpdateRPCTimeoutOrCancelledError` | No worker polling task queue |
| `WorkflowQueryFailedError` | Query handler raised |

⚠️ **Queries cannot mutate state and cannot `await`.** They re-execute on every replay; treat them as pure projections of `self`.

⚠️ **Continue-As-New is not supported inside Update handlers.** Use a wait condition in `@workflow.run` plus `workflow.all_handlers_finished` and call CAN from there. See [[Temporal DV 04 - Continue As New]].

⚠️ **Returning from `run` while handlers are mid-flight drops them.** Always `await workflow.wait_condition(workflow.all_handlers_finished)` before returning if outstanding handlers matter.

⚠️ **Validators run before handlers admit to history — handlers run after.** A successful validator + handler exception still produces a `WorkflowUpdateFailedError` on the client, but the failure *is* recorded in history; a validator rejection is *not*.

💡 **Takeaway:** Signal = async push, Query = sync read, Update = validated RPC. Use `@workflow.init` for early state, `wait_condition` to coordinate with `run`, `asyncio.Lock` to serialise concurrent handlers.
