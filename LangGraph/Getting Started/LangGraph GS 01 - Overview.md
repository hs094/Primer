# GS 01 — Overview

🔑 LangGraph is a low-level orchestration framework and runtime for building, managing, and deploying long-running, stateful agents — you wire nodes (functions) together with edges over a typed shared state.

## What it is

LangGraph isn't an agent abstraction on top of an LLM SDK — it's a graph runtime. You describe an agent as a graph: nodes are Python functions that read state and return updates, edges describe transitions, and the runtime drives the loop with checkpointing, retries, and streaming built in.

It works standalone and integrates with LangChain (model + tool abstractions), LangSmith (tracing), and the LangGraph deployment platform.

## Install

```bash
pip install -U langgraph
```

or

```bash
uv add langgraph
```

Requires Python 3.10+. See [[LangGraph GS 02 - Install LangGraph]].

## Hello world

```python
from langgraph.graph import StateGraph, MessagesState, START, END

def mock_llm(state: MessagesState):
    return {"messages": [{"role": "ai", "content": "hello world"}]}

graph = StateGraph(MessagesState)
graph.add_node(mock_llm)
graph.add_edge(START, "mock_llm")
graph.add_edge("mock_llm", END)
graph = graph.compile()

graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
```

Anatomy:

- `StateGraph(MessagesState)` — graph parameterised by a state schema. `MessagesState` is a built-in `TypedDict` with a single `messages` field that uses an append reducer.
- `add_node(mock_llm)` — node name defaults to the function's `__name__`.
- `START` / `END` — sentinel node names that mark entry and exit.
- `compile()` — turns the builder into a runnable. Required before `invoke`/`stream`.
- `invoke(initial_state)` — runs the graph to completion and returns the final state.

## Core capabilities

- **Durable execution.** Agents persist through failures and can run for extended periods, resuming from where they left off (checkpointer + `thread_id`).
- **Human-in-the-loop.** Inspect and modify agent state at any point; pause with `interrupt()` and resume with `Command(resume=...)`.
- **Comprehensive memory.** Short-term working memory across a run plus long-term memory across sessions.
- **Debugging with LangSmith.** Visualise execution paths and capture state transitions.
- **Production-ready deployment.** Scalable infrastructure for stateful, long-running workflows.

## Two APIs

LangGraph ships two ways to author the same runtime:

- **Graph API** — explicit `StateGraph` with `add_node` / `add_edge` / `add_conditional_edges`. Best for branching, parallel fan-out/fan-in, and visualisation.
- **Functional API** — `@entrypoint` / `@task` decorators. Best for sequential workflows that wrap existing procedural code.

Pick per-feature; both can coexist in one app. See [[LangGraph GS 04 - Choosing Graph vs Functional APIs]].

## Mental model

- State is your shared memory — a typed dict passed between nodes.
- Nodes return **partial updates** to state; reducers (declared on the state schema) decide how updates merge (e.g. append vs replace).
- Edges move execution forward; conditional edges or `Command(goto=...)` route dynamically.
- The runtime checkpoints state between node executions — that's what makes pause/resume and durability work.

For the design philosophy and worked example see [[LangGraph GS 05 - Thinking in LangGraph]]. For a runnable agent see [[LangGraph GS 03 - Quickstart]].

⚠️ Don't reach for LangGraph for one-shot LLM calls. The payoff is in stateful, multi-step, branching, or human-gated flows. A single prompt is just a function call.

💡 Treat LangGraph as a runtime, not a framework — you bring the LLM (via LangChain or any SDK), tools, and storage; LangGraph handles loop control, state merging, persistence, and resumability.
