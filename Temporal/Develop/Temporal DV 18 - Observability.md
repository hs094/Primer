# DV 18 — Observability

🔑 Four pillars in Temporal: **metrics** (Prometheus), **traces** (OpenTelemetry), **logs** (stdlib `logging`), and **visibility queries** (search attributes). Wire them up early — debugging long-lived Workflows without them is misery.

## 1. Metrics

Configure metrics **globally**, before any other Temporal code runs, by passing a `Runtime` to your Client:

```python
from temporalio.runtime import Runtime, TelemetryConfig, PrometheusConfig
from temporalio.client import Client

new_runtime = Runtime(
    telemetry=TelemetryConfig(
        metrics=PrometheusConfig(bind_address="0.0.0.0:9000")
    )
)
my_client = await Client.connect("my.temporal.host:7233", runtime=new_runtime)
```

This exposes a Prometheus scrape endpoint on port 9000. The full metric catalog lives in [[Temporal RF 01 - SDK Metrics]] — key ones to chart:

- `temporal_workflow_completed`, `temporal_workflow_failed`
- `temporal_activity_execution_failed`
- `temporal_workflow_task_schedule_to_start_latency` — backlog signal
- `temporal_sticky_cache_size`, `temporal_workflow_task_replay_latency`

⚠️ Pass the same `Runtime` to **every** Client and Worker in the process. Metrics initialize once globally.

## 2. Tracing (OpenTelemetry)

Install the extra:

```bash
pip install temporalio[opentelemetry]
```

Then attach the `TracingInterceptor` to Client and Worker:

```python
from temporalio.client import Client
from temporalio.contrib.opentelemetry import TracingInterceptor

client = await Client.connect(
    "localhost:7233",
    interceptors=[TracingInterceptor()],
)
```

Spans are auto-created for:

- Client calls (start, signal, query, etc.)
- Workflow invocations
- Activity executions

Trace context propagates through Workflow → Activity → Child Workflow → Nexus, giving you end-to-end traces in Jaeger/Honeycomb/Tempo.

## 3. Logging

Use stdlib `logging` — no `print`, no `structlog`-specific magic. The SDK core defaults to WARN; configure your app's root level:

```python
import logging
logging.basicConfig(level=logging.INFO)
```

### Inside Workflows

Use `workflow.logger`, **not** `logging.getLogger(__name__)`:

```python
from temporalio import workflow

@workflow.defn
class GreetingWorkflow:
    @workflow.run
    async def run(self, name: str) -> str:
        workflow.logger.info(f"Workflow input parameter: {name}")
        ...
```

⚠️ `workflow.logger` is **replay-aware** — it suppresses duplicate log lines on replay so your logs match the actual execution timeline. A naive `logging.getLogger` would re-emit on every replay.

### Inside Activities

Plain `logging.getLogger(__name__)` is fine; activities don't replay.

```python
import logging
logger = logging.getLogger(__name__)

@activity.defn
async def fetch(url: str) -> str:
    logger.info(f"fetching {url}")
    ...
```

Available levels: TRACE, DEBUG, INFO, WARN, ERROR.

## 4. Visibility (search & filter)

### List Workflows

```python
async for workflow in client.list_workflows('WorkflowType="GreetingWorkflow"'):
    print(f"Workflow: {workflow.id}")
```

The argument is a **List Filter** — SQL-like syntax over the visibility store.

### Set custom search attributes at start

```python
from temporalio.client import (
    SearchAttributeKey, SearchAttributePair, TypedSearchAttributes,
)

customer_id_key = SearchAttributeKey.for_keyword("CustomerId")
misc_data_key = SearchAttributeKey.for_text("MiscData")

handle = await client.start_workflow(
    GreetingWorkflow.run,
    id="search-attributes-workflow-id",
    task_queue="search-attributes-task-queue",
    search_attributes=TypedSearchAttributes([
        SearchAttributePair(customer_id_key, "customer_1"),
        SearchAttributePair(misc_data_key, "customer_1_data"),
    ]),
)
```

### Upsert from inside a Workflow

```python
workflow.upsert_search_attributes([
    customer_id_key.value_set("customer_2"),
])
```

### Remove

```python
workflow.upsert_search_attributes([
    customer_id_key.value_unset(),
])
```

### Search Attribute types

| Method | Type |
|---|---|
| `for_keyword(name)` | exact-match string |
| `for_text(name)` | full-text-searchable string |
| `for_int(name)` | int |
| `for_float(name)` | float |
| `for_bool(name)` | bool |
| `for_datetime(name)` | datetime |
| `for_keyword_list(name)` | list of keywords |

⚠️ Custom search attributes must be **registered** in the Namespace first via `temporal operator search-attribute create`.

## Putting it together

```python
runtime = Runtime(telemetry=TelemetryConfig(
    metrics=PrometheusConfig(bind_address="0.0.0.0:9000"),
))
client = await Client.connect(
    "localhost:7233",
    runtime=runtime,
    interceptors=[TracingInterceptor()],
)
```

Now metrics are scrape-able, traces flow to your collector, and `workflow.logger.info(...)` shows up in your log aggregator with the correct replay-aware timestamps.

## Cross-references

- [[Temporal RF 01 - SDK Metrics]] — full metric catalog
- [[Temporal DV 19 - Testing]] — Replay tests catch determinism breaks
- [[Temporal DV 22 - Debugging]] — using these signals to diagnose
- [[Temporal BP 02 - Worker Performance]] — what to watch in production

💡 **Takeaway:** Configure Prometheus metrics + OTel tracing on the `Runtime` once; use `workflow.logger` (not stdlib) inside Workflows for replay-correct logs; register search attributes and set them at start or via `upsert_search_attributes` to make Workflows queryable.
