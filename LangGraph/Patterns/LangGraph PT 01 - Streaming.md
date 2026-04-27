# PT 01 — Streaming

🔑 Streaming surfaces incremental output as the graph runs — state diffs, LLM tokens, custom progress events, or full debug traces — using a single unified `StreamPart` envelope under `version="v2"`.

## Methods

Every compiled graph exposes `stream()` (sync) and `astream()` (async). Both return iterators of `StreamPart` chunks. Stream behaviour is controlled by `stream_mode` (string or list of strings) and `version="v2"` (recommended — guarantees uniform output shape).

## Stream Modes

| Mode | Yields |
| --- | --- |
| `values` | Full state snapshot after each super-step |
| `updates` | Only the keys each node returned |
| `messages` | `(message_chunk, metadata)` tuples — LLM tokens |
| `custom` | Arbitrary payloads from `get_stream_writer()` |
| `checkpoints` | Checkpoint events (needs checkpointer) |
| `tasks` | Task start/finish events (needs checkpointer) |
| `debug` | `checkpoints + tasks` plus extra metadata |

## v2 Envelope

Every chunk is a dict with the same shape, regardless of mode or whether subgraphs are involved:

```python
{
    "type": "values" | "updates" | "messages" | "custom" | "checkpoints" | "tasks" | "debug",
    "ns": (),     # namespace tuple — non-empty for subgraph events
    "data": ...,  # payload — depends on type
}
```

Filter on `chunk["type"]` to narrow the payload. Without `version="v2"` the format varies by mode — always pass it for new code.

## Updates vs Values

```python
for chunk in graph.stream(
    {"topic": "ice cream"},
    stream_mode="updates",
    version="v2",
):
    if chunk["type"] == "updates":
        for node_name, state in chunk["data"].items():
            print(f"Node `{node_name}` updated: {state}")
```

```python
for chunk in graph.stream(
    {"topic": "ice cream"},
    stream_mode="values",
    version="v2",
):
    if chunk["type"] == "values":
        print(f"topic: {chunk['data']['topic']}, joke: {chunk['data']['joke']}")
```

`updates` is cheaper and easier to render incrementally; `values` is convenient when you need the whole state on each step.

## Token Streaming with `messages`

`messages` mode streams LLM output token-by-token from any node. Metadata includes `langgraph_node`, tags, and run info.

```python
from dataclasses import dataclass
from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START

@dataclass
class MyState:
    topic: str
    joke: str = ""

model = init_chat_model(model="gpt-5.4-mini")

def call_model(state: MyState):
    model_response = model.invoke(
        [{"role": "user", "content": f"Generate a joke about {state.topic}"}]
    )
    return {"joke": model_response.content}

graph = (
    StateGraph(MyState)
    .add_node(call_model)
    .add_edge(START, "call_model")
    .compile()
)

for chunk in graph.stream(
    {"topic": "ice cream"},
    stream_mode="messages",
    version="v2",
):
    if chunk["type"] == "messages":
        message_chunk, metadata = chunk["data"]
        if message_chunk.content:
            print(message_chunk.content, end="|", flush=True)
```

Filter by tag (`metadata["tags"]`) or node (`metadata["langgraph_node"]`). Tag a model with `["nostream"]` to silence its tokens while still letting it run. Models without streaming support take `streaming=False` (or `disable_streaming=True`).

## Custom Progress with `get_stream_writer`

For long-running tasks, push status events from inside a node or tool:

```python
from langgraph.config import get_stream_writer
from langchain.tools import tool

@tool
def query_database(query: str) -> str:
    """Query the database."""
    writer = get_stream_writer()
    writer({"data": "Retrieved 0/100 records", "type": "progress"})
    # ... work ...
    writer({"data": "Retrieved 100/100 records", "type": "progress"})
    return "some-answer"
```

Same trick lets you bridge non-LangChain LLMs into the stream — wrap their iterator and call `writer({...})` per chunk.

## Combining Modes

Pass a list to receive multiple stream types in one iterator; narrow by `chunk["type"]`:

```python
for chunk in graph.stream(
    inputs,
    stream_mode=["updates", "custom"],
    version="v2",
):
    if chunk["type"] == "updates":
        for node_name, state in chunk["data"].items():
            print(f"Node `{node_name}` updated: {state}")
    elif chunk["type"] == "custom":
        print(f"Custom event: {chunk['data']}")
```

## Subgraphs

Set `subgraphs=True` to merge subgraph events into the parent stream. The `ns` field carries the subgraph namespace; root events have `ns == ()`:

```python
for chunk in graph.stream(
    {"foo": "foo"},
    stream_mode="updates",
    subgraphs=True,
    version="v2",
):
    if chunk["type"] == "updates":
        if chunk["ns"]:
            print(f"Subgraph {chunk['ns']}: {chunk['data']}")
        else:
            print(f"Root: {chunk['data']}")
```

## Checkpoints, Tasks, Debug

`checkpoints` and `tasks` need a checkpointer (see [[LangGraph CO 05 - Persistence (Checkpointers)]]) and a `thread_id`. `debug` rolls both together with extra metadata — useful when something is misbehaving and you want one firehose.

```python
for chunk in graph.stream(
    {"topic": "ice cream"},
    stream_mode="debug",
    version="v2",
):
    if chunk["type"] == "debug":
        print(chunk["data"])
```

## Async Caveat (Python < 3.11)

Context propagation is patchy on 3.10 and below. Two adjustments:

1. Forward the incoming `RunnableConfig` to async LLM calls.
2. Inject `writer: StreamWriter` as a node argument instead of calling `get_stream_writer()`.

```python
from langgraph.types import StreamWriter

async def generate_joke(state: State, writer: StreamWriter):
    writer({"custom_key": "Streaming custom data while generating a joke"})
    return {"joke": f"This is a joke about {state['topic']}"}
```

💡 Reach for `updates` for UI state diffs, `messages` for token UX, `custom` for progress and non-LangChain LLMs, and `debug` when you need the firehose. Always pass `version="v2"` so chunk shape stays uniform.
