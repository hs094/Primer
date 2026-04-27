# DV 17 — Nexus

🔑 **Nexus** lets a Workflow in one Namespace call an operation that runs in another Namespace (or service) through a typed contract. Think "RPC across Temporal Namespaces with Workflow durability on both sides".

## Why Nexus

- **Cross-namespace calls** — caller and handler live in different Namespaces, possibly different accounts.
- **Typed contract** — service definition is a Python class with operation signatures.
- **Sync or async ops** — quick lookups (sync, ≤10s) and long-running Workflows (async).
- **Durable** — operations are Workflow events, replayable like everything else.

See [[Temporal EN 13 - Temporal Nexus]] for the concept.

## Quickstart

### 1. Service contract (`service.py`)

```python
from dataclasses import dataclass
import nexusrpc

@dataclass
class MyInput:
    name: str

@nexusrpc.service
class SayHelloNexusService:
    say_hello: nexusrpc.Operation[MyInput, str]
```

### 2. Operation handler (`handler.py`)

```python
import uuid
import nexusrpc.handler
from temporalio import nexus
from service import MyInput, SayHelloNexusService
from workflows import SayHelloWorkflow

@nexusrpc.handler.service_handler(service=SayHelloNexusService)
class SayHelloNexusServiceHandler:
    @nexus.workflow_run_operation
    async def say_hello(
        self, ctx: nexus.WorkflowRunOperationContext, input: MyInput
    ) -> nexus.WorkflowHandle[str]:
        return await ctx.start_workflow(
            SayHelloWorkflow.run,
            input.name,
            id=f"say-hello-nexus-{uuid.uuid4()}",
        )
```

### 3. Worker registration

Pass handler instances via the `nexus_service_handlers=` parameter when constructing the [[Temporal DV 15 - Worker Processes]] Worker.

### 4. Caller workflow (`caller.py`)

```python
from datetime import timedelta
from temporalio import workflow
from service import MyInput, SayHelloNexusService

NEXUS_ENDPOINT = "my-nexus-endpoint-name"

@workflow.defn
class CallerWorkflow:
    @workflow.run
    async def run(self, name: str) -> str:
        nexus_client = workflow.create_nexus_client(
            service=SayHelloNexusService,
            endpoint=NEXUS_ENDPOINT,
        )
        return await nexus_client.execute_operation(
            SayHelloNexusService.say_hello,
            MyInput(name=name),
            schedule_to_close_timeout=timedelta(seconds=10),
        )
```

### 5. Setup commands

```bash
temporal operator namespace create --namespace my-caller-namespace
temporal operator nexus endpoint create \
  --name my-nexus-endpoint-name \
  --target-namespace default \
  --target-task-queue my-task-queue
```

Expected output: `Workflow result: Hello Temporal`.

## Feature Guide

### Service definition with multiple ops

```python
from dataclasses import dataclass
import nexusrpc

@dataclass
class MyInput:
    name: str

@dataclass
class MyOutput:
    message: str

@nexusrpc.service
class MyNexusService:
    my_sync_operation: nexusrpc.Operation[MyInput, MyOutput]
    my_workflow_run_operation: nexusrpc.Operation[MyInput, MyOutput]
```

### Sync operations (≤10s)

```python
@nexusrpc.handler.service_handler(service=MyNexusService)
class MyNexusServiceHandler:
    @nexusrpc.handler.sync_operation
    async def my_sync_operation(
        self, ctx: nexusrpc.handler.StartOperationContext, input: MyInput
    ) -> MyOutput:
        return MyOutput(message=f"Hello {input.name} from sync operation!")
```

⚠️ Sync handlers **must** complete within 10 seconds. Use them for lookups, not work.

### Async (workflow-backed) operations

```python
@nexus.workflow_run_operation
async def my_workflow_run_operation(
    self, ctx: nexus.WorkflowRunOperationContext, input: MyInput
) -> nexus.WorkflowHandle[MyOutput]:
    return await ctx.start_workflow(
        WorkflowStartedByNexusOperation.run,
        input,
        id=str(uuid.uuid4()),
    )
```

The handler **starts** a Workflow and returns its handle. The caller awaits the result without knowing it's a Workflow under the hood.

### Multiple workflow args from one Nexus input

```python
@nexus.workflow_run_operation
async def hello(
    self, ctx: nexus.WorkflowRunOperationContext, input: HelloInput
) -> nexus.WorkflowHandle[HelloOutput]:
    return await ctx.start_workflow(
        HelloHandlerWorkflow.run,
        args=[input.name, input.language],
        id=str(uuid.uuid4()),
    )
```

### Caller patterns

```python
@workflow.defn
class CallerWorkflow:
    @workflow.run
    async def run(self, name: str) -> tuple[MyOutput, MyOutput]:
        nexus_client = workflow.create_nexus_client(
            service=MyNexusService,
            endpoint=NEXUS_ENDPOINT,
        )
        # execute = start + await
        wf_result = await nexus_client.execute_operation(
            MyNexusService.my_workflow_run_operation,
            MyInput(name),
        )
        # start, then await the handle
        sync_handle = await nexus_client.start_operation(
            MyNexusService.my_sync_operation,
            MyInput(name),
        )
        sync_result = await sync_handle
        return sync_result, wf_result
```

### Errors

| Type | Use |
|---|---|
| `nexusrpc.OperationError` | Operation failed, no retry |
| `nexusrpc.HandlerError` | Specify retryable / non-retryable per Nexus spec |
| `temporalio.exceptions.NexusOperationError` | Raised in caller Workflow when op fails |

### Cancellation

Async ops support `handle.cancel()`. Cancellation policies:

- `ABANDON` — no cancel request sent
- `TRY_CANCEL` — fire-and-forget cancel
- `WAIT_REQUESTED` — wait for ack
- `WAIT_COMPLETED` (default) — wait for op to finish

### Inside the handler

```python
from temporalio import nexus

client = nexus.client()  # Worker's Temporal Client
info = nexus.info()      # current operation info
```

### Observability

Sync ops emit: `NexusOperationScheduled` → `NexusOperationCompleted`.
Async ops emit: `NexusOperationScheduled` → `NexusOperationStarted` → `NexusOperationCompleted`.

Inspect via Web UI (`http://localhost:8233`) or:

```bash
temporal workflow describe -w <ID>
temporal workflow show -w <ID>
```

### Temporal Cloud setup

```bash
brew install temporalio/brew/tcld
tcld gen ca --org YOUR_ORG --validity-period 1y \
  --ca-cert ca.pem --ca-key ca.key
tcld namespace create --namespace my-caller-namespace \
  --cloud-provider aws --region us-west-2 \
  --ca-certificate-file path/to/ca.pem --retention-days 1
tcld nexus endpoint create --name my-endpoint \
  --target-task-queue handler-queue \
  --target-namespace target-ns.account \
  --allow-namespace caller-ns.account
```

## Cross-references

- [[Temporal EN 13 - Temporal Nexus]] — concept
- [[Temporal DV 03 - Child Workflows]] — same-namespace alternative
- [[Temporal DV 15 - Worker Processes]] — registering handlers

💡 **Takeaway:** Nexus = typed RPC across Namespaces with sync ops for lookups (≤10s) and workflow-run ops for long work. Define a service, implement handlers, configure an endpoint, and call from any Workflow with `workflow.create_nexus_client(...)`.
