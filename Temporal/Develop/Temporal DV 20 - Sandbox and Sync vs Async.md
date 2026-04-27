# DV 20 — Sandbox and Sync vs Async

🔑 Two foot-gun zones in the Python SDK: the **Workflow Sandbox** (which intercepts imports to prevent non-determinism) and the **sync-vs-async Activity choice** (which decides whether your event loop survives a blocking call). Get both right or workers freeze in production.

## Sandbox

### How it works

When a Workflow starts, the SDK imports your Workflow module into a **fresh sandbox** using `exec` and isolated globals. The sandbox:

- Lets stdlib + Temporal modules pass through (assumed side-effect-free).
- Wraps other modules in proxies that block known non-deterministic calls (`time.time()`, `random.random()`, `os.environ`, network I/O, etc.) at both import and runtime.
- Re-imports the module per Workflow run so module-level state can't leak.

This catches accidental non-determinism early instead of silently corrupting Workflow histories.

### Skipping the sandbox

**For a code block:**
```python
with temporalio.workflow.unsafe.sandbox_unrestricted():
    # your code
```

**For an entire Workflow:**
```python
@workflow.defn(sandboxed=False)
class MyWorkflow:
    ...
```

**For the whole Worker:**
```python
from temporalio.worker.workflow_sandbox import UnsandboxedWorkflowRunner

worker = Worker(
    ...,
    workflow_runner=UnsandboxedWorkflowRunner(),
)
```

⚠️ Disabling the sandbox shifts the burden of determinism entirely onto you. Prefer passthrough modules.

### Allowing safe modules through (passthrough)

Passthrough = "import once outside the sandbox, share the instance". Faster, but only safe for **deterministic, side-effect-free** modules.

**Per-import:**
```python
from temporalio import workflow

with workflow.unsafe.imports_passed_through():
    import pydantic

@workflow.defn
class MyWorkflow:
    ...
```

**Worker-wide:**
```python
from temporalio.worker import Worker
from temporalio.worker.workflow_sandbox import (
    SandboxedWorkflowRunner, SandboxRestrictions,
)

worker = Worker(
    ...,
    workflow_runner=SandboxedWorkflowRunner(
        restrictions=SandboxRestrictions.default.with_passthrough_modules("pydantic")
    ),
)
```

### Allowing a specific child of a flagged module

Some stdlib members (`datetime.date.today`) are blocked by default but you may have a deterministic use case:

```python
import dataclasses
from temporalio.worker.workflow_sandbox import SandboxedWorkflowRunner, SandboxRestrictions

my_restrictions = dataclasses.replace(
    SandboxRestrictions.default,
    invalid_module_members=SandboxRestrictions.invalid_module_members_default.with_child_unrestricted(
        "datetime", "date", "today",
    ),
)
worker = Worker(..., workflow_runner=SandboxedWorkflowRunner(restrictions=my_restrictions))
```

Or via `SandboxMatcher` for finer control:

```python
from temporalio.worker.workflow_sandbox import SandboxMatcher

my_restrictions = dataclasses.replace(
    SandboxRestrictions.default,
    invalid_module_members=SandboxRestrictions.invalid_module_members_default | SandboxMatcher(
        children={"datetime": SandboxMatcher(use={"date"})},
    ),
)
```

### Import notification policy

Get warned when modules slip through unintentionally:

```python
worker = Worker(
    ...,
    workflow_runner=SandboxedWorkflowRunner(
        restrictions=SandboxRestrictions.default.with_import_notification_policy(
            workflow.SandboxImportNotificationPolicy.WARN_ON_DYNAMIC_IMPORT |
            workflow.SandboxImportNotificationPolicy.WARN_ON_UNINTENTIONAL_PASSTHROUGH
        )
    ),
)
```

`WARN_ON_DYNAMIC_IMPORT` is on by default. Silence per-import with `SILENT`.

---

## Sync vs Async

### Three Activity shapes

| Style | Executor | When |
|---|---|---|
| `async def` | asyncio loop | I/O with **async-safe** libs (`aiohttp`, `httpx`, `asyncpg`) |
| `def` (threaded) | `ThreadPoolExecutor` | I/O with **blocking** libs (`requests`, `psycopg2`) — **default choice** |
| `def` (process) | `ProcessPoolExecutor` + `SyncManager` | CPU-bound work that needs to dodge the GIL |

### The core trap

> The event loop can only pass control when `await` runs. Any blocking call (file read, sync HTTP, `time.sleep`) freezes the **entire** Worker — including the polling loop that fetches more tasks.

A single `requests.get(...)` inside an `async def` Activity converts your concurrent Worker into a sequential one. Worse, it can deadlock if the Activity needs another async resource.

### Recommended default: synchronous Activities

```python
import urllib.parse
import requests
from temporalio import activity

class TranslateActivities:
    @activity.defn
    def greet_in_spanish(self, name: str) -> str:
        return self.call_service("get-spanish-greeting", name)

    def call_service(self, stem: str, name: str) -> str:
        base = f"http://localhost:9999/{stem}"
        url = f"{base}?name={urllib.parse.quote(name)}"
        response = requests.get(url)
        return response.text
```

Worker config — sync Activities **require** an `activity_executor`:

```python
from concurrent.futures import ThreadPoolExecutor
from temporalio.worker import Worker

with ThreadPoolExecutor(max_workers=42) as executor:
    worker = Worker(
        ...,
        activity_executor=executor,
    )
```

### Async Activities (when you have async-safe libs)

```python
import aiohttp
import urllib.parse
from temporalio import activity
from temporalio.exceptions import ApplicationError

class TranslateActivities:
    def __init__(self, session: aiohttp.ClientSession):
        self.session = session

    @activity.defn
    async def greet_in_spanish(self, name: str) -> str:
        return await self.call_service("get-spanish-greeting", name)

    async def call_service(self, stem: str, name: str) -> str:
        base = f"http://localhost:9999/{stem}"
        url = f"{base}?name={urllib.parse.quote(name)}"
        async with self.session.get(url) as response:
            translation = await response.text()
            if response.status >= 400:
                raise ApplicationError(
                    f"HTTP Error {response.status}: {translation}",
                    non_retryable=response.status < 500,
                )
            return translation
```

### Escape hatches when you must block in async

```python
import asyncio

# preferred
result = await asyncio.to_thread(blocking_call, arg)

# or explicit executor
loop = asyncio.get_running_loop()
result = await loop.run_in_executor(None, blocking_call, arg)
```

### Workflow execution model

Despite using `async def`, **Workflow Definitions don't run on the asyncio loop**. They run on a dedicated `workflow_task_executor` thread pool with a deterministic "Workflow event loop" — so `asyncio.sleep` won't work; use `workflow.sleep` instead.

### CPU and the GIL

One Python process = effectively one core for CPU-bound work. Options:

1. **Multiple Worker processes** (horizontal scale).
2. **`ProcessPoolExecutor` for sync Activities** — work runs in worker subprocesses, GIL doesn't matter.

### Debug tip

If an async Activity is misbehaving in mysterious ways (timeouts, hung Workers), **rewrite it as sync** with a `ThreadPoolExecutor`. Most "weird async" bugs are blocking calls in disguise.

## Cross-references

- [[Temporal DV 15 - Worker Processes]] — `activity_executor` plumbing
- [[Temporal DV 19 - Testing]] — sandbox can affect mocks
- [[Temporal DV 21 - Data Handling]] — Pydantic needs passthrough
- [[Temporal BP 02 - Worker Performance]] — sizing executors

💡 **Takeaway:** Keep the sandbox on, passthrough only known-deterministic modules. For Activities, **default to sync + ThreadPoolExecutor**; reach for async only when every library on the call path is async-safe; use `ProcessPoolExecutor` or multiple processes for CPU-bound work.
