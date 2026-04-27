# DV 16 — Temporal Client

🔑 The **Client** is your gateway to the Temporal Service from outside Workflows. Use it to start Workflows, send Signals/Updates, run Queries, fetch results, and list executions. One persistent client per process; reuse it.

## When to use the Client

- **Starters / API handlers**: launch Workflows from web requests, CLIs, cron jobs.
- **Inside Activities**: query/signal other Workflows, list executions, manage Schedules.
- **Tests**: drive Workflows under [[Temporal DV 19 - Testing]].

⚠️ Clients **cannot** be initialized inside Workflows. Workflow code must stay deterministic and side-effect-free; use Activities to reach the Client.

## Connecting (local dev)

Defaults are `127.0.0.1:7233` and namespace `default`:

```python
from temporalio.client import Client

async def main():
    client = await Client.connect("localhost:7233")
```

The Client is async, persistent, and thread-safe. Build it once and pass it around.

### Via env vars

```bash
export TEMPORAL_ADDRESS="localhost:7233"
export TEMPORAL_NAMESPACE="default"
```

### Via TOML config

```python
import asyncio
from pathlib import Path
from temporalio.client import Client
from temporalio.envconfig import ClientConfig

async def main():
    config_file = Path(__file__).parent / "config.toml"
    connect_config = ClientConfig.load_client_connect_config(
        config_file=str(config_file)
    )
    await Client.connect(**connect_config)
```

## Connecting to Temporal Cloud

Cloud requires the namespace ID combined with account ID (`<namespace_id>.<account_id>`) and either an API key or mTLS certificates.

### API key (TOML)

```toml
[profile.staging]
address = "your-namespace.a1b2c.tmprl.cloud:7233"
namespace = "your-namespace"
api_key = "your-api-key-here"
```

### mTLS (TOML)

```toml
[profile.staging]
address = "your-namespace.a1b2c.tmprl.cloud:7233"
namespace = "your-namespace"
tls_client_cert_data = "your-cert-data"
tls_client_key_path = "your-key-path"
```

### mTLS in code

```python
from temporalio.client import Client, TLSConfig

async def main():
    with open("client-cert.pem", "rb") as f:
        client_cert = f.read()
    with open("client-private-key.pem", "rb") as f:
        client_private_key = f.read()

    client = await Client.connect(
        "your-custom-namespace.tmprl.cloud:7233",
        namespace="<your-custom-namespace>.<account-id>",
        tls=TLSConfig(
            client_cert=client_cert,
            client_private_key=client_private_key,
        ),
    )
```

## Starting Workflows

```python
async def main():
    client = await Client.connect("localhost:7233")
    result = await client.execute_workflow(
        YourWorkflow.run,
        "your name",
        id="your-workflow-id",
        task_queue="your-task-queue",
    )
    print(f"Result: {result}")
```

| Method | Use it for |
|---|---|
| `execute_workflow(...)` | Start **and** await the result in one call |
| `start_workflow(...)` | Start, get a handle back, await later (or never) |

Required params: `id` (your business identifier — order ID, customer ID — never a UUID for traceable work) and `task_queue` (must match a [[Temporal DV 15 - Worker Processes]] poller).

## Retrieving results / handles

```python
async def main():
    client = await Client.connect("localhost:7233")
    handle = client.get_workflow_handle(workflow_id="your-workflow-id")
    results = await handle.result()
    print(f"Result: {results}")
```

- `get_workflow_handle(workflow_id)` — handle for an existing Workflow.
- `get_workflow_handle_for(YourWorkflow.run, workflow_id)` — typed handle (better IDE support, return type is known).
- `handle.describe()` — current status, run ID, history length, etc.

## What else the Client does

| Operation | Method |
|---|---|
| Signal | `handle.signal(WF.signal, payload)` — see [[Temporal DV 07 - Message Passing]] |
| Query | `handle.query(WF.query)` — see [[Temporal DV 07 - Message Passing]] |
| Update | `handle.execute_update(WF.update, payload)` — see [[Temporal DV 07 - Message Passing]] |
| Cancel | `handle.cancel()` — see [[Temporal DV 05 - Workflow Cancellation]] |
| Terminate | `handle.terminate(reason="...")` |
| List | `client.list_workflows("WorkflowType='Foo'")` — see [[Temporal DV 18 - Observability]] |

## Custom Data Converter

Pass `data_converter=` at connect time to plug in encryption, compression, or Pydantic JSON. See [[Temporal DV 21 - Data Handling]].

```python
client = await Client.connect(
    "localhost:7233",
    data_converter=my_data_converter,
)
```

## Cross-references

- [[Temporal EN 02 - Temporal SDKs]] — what the SDK gives you
- [[Temporal DV 15 - Worker Processes]] — the other half
- [[Temporal CL 02 - Setup CLI]] — local Service to point the Client at
- [[Temporal DV 21 - Data Handling]] — custom converters

💡 **Takeaway:** One Client per process, configured once. It starts Workflows, fetches results, and routes Signals/Updates/Queries. Inside Workflows you don't need a Client — for cross-Workflow calls reach for [[Temporal DV 17 - Nexus]] or [[Temporal DV 03 - Child Workflows]] instead.
