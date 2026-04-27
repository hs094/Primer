# GS 04 — Choosing Graph vs Functional APIs

🔑 Graph API for explicit branching, fan-out/fan-in, and visualisation; Functional API for sequential flows that wrap existing procedural code — they share the same runtime (persistence, streaming, HITL) and can be mixed in one app.

## Quick rule of thumb

| You want… | Pick |
|---|---|
| Multiple decision points, easy to visualise | Graph API |
| Explicit shared state across many nodes | Graph API |
| Parallel branches that fan in to a join | Graph API |
| Wrap existing procedural code with minimal refactor | Functional API |
| Standard `if`/`else`/`for` control flow | Functional API |
| Rapid prototyping, no state schema yet | Functional API |

Both APIs give you the same persistence, streaming, and human-in-the-loop primitives — the choice is about **how you express control flow**, not what's available.

## Graph API — when

Use `StateGraph` when:

- The workflow has multiple decision points that depend on various conditions — the Graph API makes branches explicit and easy to visualise.
- Multiple components share data — explicit state is clearer than threading kwargs through functions.
- You need parallel processing where multiple operations run simultaneously and merge results.
- You want a Mermaid diagram of the agent for cross-team docs.

**Branching example:**

```python
from langgraph.graph import StateGraph
from typing import TypedDict

class AgentState(TypedDict):
    messages: list
    current_tool: str
    retry_count: int

def should_continue(state):
    if state["retry_count"] > 3:
        return "end"
    elif state["current_tool"] == "search":
        return "process_search"
    else:
        return "call_llm"

workflow = StateGraph(AgentState)
workflow.add_node("call_llm", call_llm_node)
workflow.add_node("process_search", search_node)
workflow.add_conditional_edges("call_llm", should_continue)
```

**Parallel fan-out / fan-in:**

```python
workflow.add_node("fetch_news", fetch_news)
workflow.add_node("fetch_weather", fetch_weather)
workflow.add_node("fetch_stocks", fetch_stocks)
workflow.add_node("combine_data", combine_all_data)

workflow.add_edge(START, "fetch_news")
workflow.add_edge(START, "fetch_weather")
workflow.add_edge(START, "fetch_stocks")

workflow.add_edge("fetch_news", "combine_data")
workflow.add_edge("fetch_weather", "combine_data")
workflow.add_edge("fetch_stocks", "combine_data")
```

All three fetchers run in parallel; `combine_data` only fires once all three complete.

## Functional API — when

Use `@entrypoint` / `@task` when:

- You're integrating existing procedural code with minimal changes.
- The flow is mostly sequential with a handful of `if`/`else` branches.
- You want to test ideas quickly without defining a state schema.
- State is naturally function-scoped — local variables, not a shared dict.

**Wrapping existing code:**

```python
from langgraph.func import entrypoint, task

@task
def process_user_input(user_input: str) -> dict:
    return {"processed": user_input.lower().strip()}

@entrypoint(checkpointer=checkpointer)
def workflow(user_input: str) -> str:
    processed = process_user_input(user_input).result()

    if "urgent" in processed["processed"]:
        response = handle_urgent_request(processed).result()
    else:
        response = handle_normal_request(processed).result()

    return response
```

**Linear with HITL:**

```python
@entrypoint(checkpointer=checkpointer)
def essay_workflow(topic: str) -> dict:
    outline = create_outline(topic).result()

    if len(outline["points"]) < 3:
        outline = expand_outline(outline).result()

    draft = write_draft(outline).result()
    feedback = interrupt({"draft": draft, "action": "Please review"})

    if feedback == "approve":
        final_essay = draft
    else:
        final_essay = revise_essay(draft, feedback).result()

    return {"essay": final_essay}
```

`interrupt(...)` works the same way it does inside a graph node — pauses the run, persists state, resumes via `Command(resume=...)`.

## Hybrid

Both APIs coexist. Drive top-level coordination with a graph; use a functional sub-workflow for a self-contained sequential pipeline:

```python
from langgraph.graph import StateGraph
from langgraph.func import entrypoint

coordination_graph = StateGraph(CoordinationState)
coordination_graph.add_node("orchestrator", orchestrator_node)
coordination_graph.add_node("agent_a", agent_a_node)
coordination_graph.add_node("agent_b", agent_b_node)

@entrypoint()
def data_processor(raw_data: dict) -> dict:
    cleaned = clean_data(raw_data).result()
    transformed = transform_data(cleaned).result()
    return transformed

def orchestrator_node(state):
    processed_data = data_processor.invoke(state["raw_data"])
    return {"processed_data": processed_data}
```

## Migration path

A workflow that started functional can graduate to a graph as branching grows. Start simple — promote to `StateGraph` when conditional edges or parallelism appear. The runtime semantics are the same; you're swapping notation, not behaviour.

⚠️ Functional `@task` calls return **futures**, not values. Forgetting `.result()` is a common bug — you'll silently pass a future into the next call.

💡 Default to Functional for sequential glue; reach for Graph when you have multiple routing decisions or want a diagram. Both compile to the same engine — see [[LangGraph GS 05 - Thinking in LangGraph]] for the underlying mental model.
