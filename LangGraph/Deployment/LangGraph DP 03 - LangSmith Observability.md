# DP 03 — LangSmith Observability

🔑 Two env vars (`LANGSMITH_TRACING=true`, `LANGSMITH_API_KEY=...`) and every node, tool call, and LLM hop becomes a searchable, replayable trace. Tags + metadata are the join keys that make those traces useful in production.

## Enable tracing

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=lsv2_pt_...
```

With these set, all LangChain/LangGraph runnables auto-instrument. Traces land in a project named `default`.

## Pick a project

```bash
export LANGSMITH_PROJECT=email-agent-prod   # static
```

```python
import langsmith as ls

with ls.tracing_context(project_name="email-agent-test", enabled=True):
    response = agent.invoke({"messages": [{"role": "user", "content": "Hi"}]})
```

## Selective tracing

`tracing_context` flips tracing on/off for a block:

```python
with ls.tracing_context(enabled=True):
    agent.invoke({"messages": [{"role": "user", "content": "Send test email"}]})

# Outside the block, only traced if LANGSMITH_TRACING=true globally
```

`enabled=False` suppresses traces inside a block.

## Tags + metadata

Threaded through `RunnableConfig` on `.invoke` / `.astream`:

```python
response = agent.invoke(
    {"messages": [{"role": "user", "content": "Send welcome email"}]},
    config={
        "tags": ["production", "email-assistant", "v1.0"],
        "metadata": {
            "user_id": "user_123",
            "session_id": "session_456",
            "environment": "production",
        },
        "run_name": "welcome-email-flow",
    },
)
```

Or hoisted into the `tracing_context`:

```python
with ls.tracing_context(
    project_name="email-agent-test",
    enabled=True,
    tags=["production", "email-assistant", "v1.0"],
    metadata={"user_id": "user_123", "session_id": "session_456"},
):
    response = agent.invoke({"messages": [...]})
```

`tags` are searchable filter chips in the LangSmith UI. `metadata` is structured payload — use it for `user_id`, `tenant_id`, deploy SHA, model name, anything you'd want to group/filter by later. `run_name` overrides the default node name shown at the top of the trace tree.

## Run trees

Each invocation produces a tree: parent run = the graph, children = nodes, grandchildren = tool calls and LLM requests. Inputs, outputs, latency, token counts, and exceptions are captured per node. Studio (see [[LangGraph DP 04 - LangSmith Studio|DP 04]]) renders the same tree with a graph view on top.

## Mask sensitive data

Anonymizers run client-side before payloads leave the process:

```python
from langchain_core.tracers.langchain import LangChainTracer
from langsmith import Client
from langsmith.anonymizer import create_anonymizer

anonymizer = create_anonymizer([
    {"pattern": r"\b\d{3}-?\d{2}-?\d{4}\b", "replace": "<ssn>"},
])
tracer = LangChainTracer(client=Client(anonymizer=anonymizer))
graph = builder.compile().with_config({"callbacks": [tracer]})
```

Patterns can be regex or callables. Apply when PII risks landing in prompts.

## Hard-disable for privacy mode

```bash
LANGSMITH_TRACING=false
```

Useful in [[LangGraph DP 01 - Run a Local Server|`langgraph dev`]] when you want the in-memory server but no telemetry to LangSmith.

## What gets captured automatically

LangChain runnables (chat models, tools, retrievers, embeddings), LangGraph node invocations + state diffs, streaming events, and exceptions with full stack + state at failure.

## Manual `@traceable`

For non-LangChain code paths you want to appear in the same tree:

```python
from langsmith import traceable

@traceable(run_type="tool", name="lookup_customer")
def lookup_customer(customer_id: str) -> dict:
    return db.fetch(customer_id)
```

Calls inside an active trace become child runs automatically.

## Wiring in deployments

Any deployment created via [[LangGraph DP 02 - LangSmith Deployment|`langgraph deploy` / `langgraph up`]] inherits these env vars. Bake them into your container env (or LangSmith Deployments secret store) and traces start flowing without code changes.

## Trace-driven workflow

Ship with tags like `["v1.0", "shadow"]` → filter LangSmith → Project → tag chip → click a regressed run → open in [[LangGraph DP 04 - LangSmith Studio|Studio]] for time-travel replay → curate failing runs into a dataset → feed into [[LangGraph TS 01 - Testing|`pytest --langsmith`]] regression suite.

💡 Tags + metadata are cheap; add them at every entry point. The investment pays off the first time you need to answer "what did user 123 see at 03:14 UTC?"
