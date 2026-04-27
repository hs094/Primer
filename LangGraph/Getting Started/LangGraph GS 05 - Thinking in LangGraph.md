# GS 05 — Thinking in LangGraph

🔑 Decompose the agent into discrete nodes that read and update a typed shared state; keep state as raw data, format prompts on demand, and use `Command` and `interrupt()` to make routing and human checkpoints explicit.

## The four primitives

- **Nodes** — Python functions that take the current state and return updates to it. One responsibility each.
- **State** — a typed dict shared across all nodes. The agent's working memory.
- **Edges / `Command`** — how control moves between nodes. Static edges are declared in the builder; dynamic routing returns a `Command(goto=...)` from inside a node.
- **Checkpointer** — persists state between node executions. Required for `interrupt()` and resume.

## Node types

A node is just a function, but it usually plays one of these roles:

- **LLM node** — understanding, analysis, generation, reasoning.
- **Data node** — retrieving from a vector store, DB, or API.
- **Action node** — performing external operations (sending email, writing a file).
- **User input node** — collecting human intervention via `interrupt()`.

## State design — store raw data, format on demand

The single most important rule: **state stores raw data, not formatted text**. Format prompts inside the node that needs them.

```python
from typing import TypedDict, Literal

class EmailClassification(TypedDict):
    intent: Literal["question", "bug", "billing", "feature", "complex"]
    urgency: Literal["low", "medium", "high", "critical"]
    topic: str
    summary: str

class EmailAgentState(TypedDict):
    email_content: str
    sender_email: str
    email_id: str
    classification: EmailClassification | None
    search_results: list[str] | None
    customer_history: dict | None
    draft_response: str | None
    messages: list[str] | None
```

Why raw:

- Different nodes can format the same data differently.
- Prompt template changes don't require state migration.
- Inspecting state during debugging stays readable.

## Routing with `Command`

Instead of declaring all edges up-front, a node can return a `Command` that updates state and picks the next node. The `Literal[...]` type hint documents the destinations.

```python
from langgraph.types import Command

def classify_intent(state: EmailAgentState) -> Command[Literal["search_documentation", "human_review", "draft_response", "bug_tracking"]]:
    structured_llm = llm.with_structured_output(EmailClassification)
    classification = structured_llm.invoke(f"... {state['email_content']} ...")

    if classification['intent'] == 'billing' or classification['urgency'] == 'critical':
        goto = "human_review"
    elif classification['intent'] in ['question', 'feature']:
        goto = "search_documentation"
    elif classification['intent'] == 'bug':
        goto = "bug_tracking"
    else:
        goto = "draft_response"

    return Command(update={"classification": classification}, goto=goto)
```

This keeps routing logic next to the data that drives it, instead of scattered across `add_conditional_edges` callbacks.

## Error handling — by who fixes it

Categorise errors by **who** can resolve them:

| Category | Handler | Approach |
|---|---|---|
| Transient (network, rate limit) | System | `RetryPolicy` |
| LLM-recoverable (tool failure) | LLM | Store error in state, loop back |
| User-fixable (missing info) | Human | `interrupt()` |
| Unexpected | Developer | Let it bubble up |

**Transient** — declare a retry on the node:

```python
from langgraph.types import RetryPolicy

workflow.add_node(
    "search_documentation",
    search_documentation,
    retry_policy=RetryPolicy(max_attempts=3, initial_interval=1.0)
)
```

**LLM-recoverable** — write the error back into state so the LLM can react:

```python
def execute_tool(state: State) -> Command[Literal["agent", "execute_tool"]]:
    try:
        result = run_tool(state['tool_call'])
        return Command(update={"tool_result": result}, goto="agent")
    except ToolError as e:
        return Command(update={"tool_result": f"Tool error: {str(e)}"}, goto="agent")
```

**User-fixable** — pause and ask:

```python
def lookup_customer_history(state: State) -> Command[Literal["draft_response"]]:
    if not state.get('customer_id'):
        user_input = interrupt({
            "message": "Customer ID needed",
            "request": "Please provide the customer's account ID"
        })
        return Command(
            update={"customer_id": user_input['customer_id']},
            goto="lookup_customer_history"
        )
    customer_data = fetch_customer_history(state['customer_id'])
    return Command(update={"customer_history": customer_data}, goto="draft_response")
```

**Unexpected** — don't swallow:

```python
def send_reply(state: EmailAgentState):
    try:
        email_service.send(state["draft_response"])
    except Exception:
        raise
```

## Human-in-the-loop with `interrupt()`

`interrupt()` pauses execution indefinitely while preserving full state. Combined with a checkpointer, the run can resume days later — identified by `thread_id` in the run config.

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt, Command

memory = MemorySaver()
app = workflow.compile(checkpointer=memory)

config = {"configurable": {"thread_id": "customer_123"}}
result = app.invoke(initial_state, config)
# graph paused at human_review; result contains __interrupt__ payload

human_response = Command(resume={"approved": True, "edited_response": "..."})
final_result = app.invoke(human_response, config)
```

⚠️ `interrupt()` must be the **first** thing in any node that uses it — code before the interrupt re-runs on resume because the node restarts from the top.

## Workflow design checklist

1. Map discrete steps as nodes; sketch the connections.
2. Identify the operation type for each step (LLM, data, action, human).
3. Design state schema with only **necessary, raw** data.
4. Implement nodes; pick the right error category for each.
5. Wire essential edges; use `Command` for dynamic routing.

💡 The state schema is your contract. Keep it small, keep it raw, and let each node decide how to format what it needs — that's what keeps a LangGraph agent maintainable as it grows. Back to [[LangGraph GS 01 - Overview]].
