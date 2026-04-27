# GS 05 — Set Up Local Python

🔑 **Install the SDK, run `temporal server start-dev`, run a Worker, fire a Workflow — the whole loop in five files.**

## Prereqs
Python 3.13+ (docs use 3.13.3). Check:

```bash
python3 -V
```

## Project bootstrap
The official docs use `pip` + `venv`:

```bash
mkdir temporal-project
cd temporal-project
python3 -m venv env
source env/bin/activate
pip install temporalio
```

`temporalio` is the only package needed for SDK work.

## Install the Temporal CLI
The CLI ships a local dev server with an in-memory database — no external Cassandra / Postgres required.

```bash
brew install temporal
```

Linux / Windows: download the platform archive from `temporal.download` and put `temporal` on your `PATH`. CLI reference: [[Temporal CL 02 - Setup CLI]].

## Start the dev server
```bash
temporal server start-dev
```

This boots:
- gRPC frontend on `localhost:7233`
- Web UI on `http://localhost:8233`
- Default Namespace pre-created

Override the UI port with `--ui-port`. Leave this terminal running.

## The four files
Hello-world Workflow, verbatim from the docs.

### `activities.py`
```python
from temporalio import activity

@activity.defn
async def greet(name: str) -> str:
    return f"Hello {name}"
```

### `workflows.py`
```python
from datetime import timedelta
from temporalio import workflow

with workflow.unsafe.imports_passed_through():
    from activities import greet

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

The `imports_passed_through()` context bypasses the SDK's import sandbox for known-safe modules — needed because Workflow files run under deterministic import rules.

### `worker.py`
```python
import asyncio
from temporalio.client import Client
from temporalio.worker import Worker
from temporalio import workflow

with workflow.unsafe.imports_passed_through():
    from workflows import SayHelloWorkflow
    from activities import greet

async def main():
    client = await Client.connect("localhost:7233")
    worker = Worker(
        client,
        task_queue="my-task-queue",
        workflows=[SayHelloWorkflow],
        activities=[greet],
    )
    print("Worker started.")
    await worker.run()

if __name__ == "__main__":
    asyncio.run(main())
```

### `starter.py`
```python
import asyncio
import uuid
from temporalio.client import Client

async def main():
    client = await Client.connect("localhost:7233")
    result = await client.execute_workflow(
        "SayHelloWorkflow",
        "Temporal",
        id=f"say-hello-workflow-{uuid.uuid4()}",
        task_queue="my-task-queue",
    )
    print("Workflow result:", result)

if __name__ == "__main__":
    asyncio.run(main())
```

## Run it
Three terminals total — server, worker, starter.

```bash
# terminal 1: dev server already running
temporal server start-dev

# terminal 2: worker
source env/bin/activate
python worker.py
# -> Worker started.

# terminal 3: trigger
source env/bin/activate
python starter.py
# -> Workflow result: Hello Temporal
```

Open `http://localhost:8233` and you'll see the Workflow Execution with its Event History. See [[Temporal DV 16 - Temporal Client]] for client patterns.

## What just happened
1. `starter.py` told the Service to start `SayHelloWorkflow` on `my-task-queue`.
2. The Service durably recorded `WorkflowExecutionStarted`.
3. The Worker polled the queue, got a Workflow Task, ran the Workflow code, and returned a command: "schedule activity `greet`."
4. The Service recorded the command, scheduled the Activity Task, sent it back to the Worker.
5. The Worker ran `greet`, returned the result. Service recorded `ActivityTaskCompleted`.
6. Worker resumed the Workflow with the result, returned the final value. Service recorded `WorkflowExecutionCompleted`.
7. `starter.py` got the result back over the same client connection.

⚠️ The dev server uses an in-memory DB — restarting it wipes all Workflow state. Fine for local development, never for anything you want to keep. For persistence, run a real Temporal Service with Postgres / Cassandra.

💡 **Takeaway:** Five commands, four files, one terminal each — and you've run a durable Workflow end-to-end on your laptop.
