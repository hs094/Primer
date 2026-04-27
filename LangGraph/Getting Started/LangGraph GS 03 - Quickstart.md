# GS 03 — Quickstart

🔑 The same calculator agent — model + three tools + a tool-calling loop — written first as a `StateGraph` and then with `@entrypoint`/`@task`; both compile to the same runtime.

## Setup

```bash
pip install -U langgraph langchain langchain-anthropic
export ANTHROPIC_API_KEY=sk-ant-...
```

## Tools and model (shared by both APIs)

```python
from langchain.tools import tool
from langchain.chat_models import init_chat_model

model = init_chat_model(
    "claude-sonnet-4-6",
    temperature=0
)

@tool
def multiply(a: int, b: int) -> int:
    """Multiply `a` and `b`."""
    return a * b

@tool
def add(a: int, b: int) -> int:
    """Adds `a` and `b`."""
    return a + b

@tool
def divide(a: int, b: int) -> float:
    """Divide `a` and `b`."""
    return a / b

tools = [add, multiply, divide]
tools_by_name = {tool.name: tool for tool in tools}
model_with_tools = model.bind_tools(tools)
```

## Graph API

**State** — `messages` uses `operator.add` so node returns are appended, not overwritten:

```python
from langchain.messages import AnyMessage
from typing_extensions import TypedDict, Annotated
import operator

class MessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], operator.add]
    llm_calls: int
```

**LLM node:**

```python
from langchain.messages import SystemMessage

def llm_call(state: dict):
    return {
        "messages": [
            model_with_tools.invoke(
                [SystemMessage(content="You are a helpful assistant tasked with performing arithmetic on a set of inputs.")]
                + state["messages"]
            )
        ],
        "llm_calls": state.get('llm_calls', 0) + 1
    }
```

**Tool node** — executes every tool call on the last message:

```python
from langchain.messages import ToolMessage

def tool_node(state: dict):
    result = []
    for tool_call in state["messages"][-1].tool_calls:
        tool = tools_by_name[tool_call["name"]]
        observation = tool.invoke(tool_call["args"])
        result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
    return {"messages": result}
```

**Conditional edge** — keep looping while the LLM keeps calling tools:

```python
from typing import Literal
from langgraph.graph import StateGraph, START, END

def should_continue(state: MessagesState) -> Literal["tool_node", END]:
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tool_node"
    return END
```

**Wire and compile:**

```python
agent_builder = StateGraph(MessagesState)
agent_builder.add_node("llm_call", llm_call)
agent_builder.add_node("tool_node", tool_node)

agent_builder.add_edge(START, "llm_call")
agent_builder.add_conditional_edges("llm_call", should_continue, ["tool_node", END])
agent_builder.add_edge("tool_node", "llm_call")

agent = agent_builder.compile()
```

**Run:**

```python
from langchain.messages import HumanMessage

messages = [HumanMessage(content="Add 3 and 4.")]
result = agent.invoke({"messages": messages})
for m in result["messages"]:
    m.pretty_print()
```

Visualise the graph (Jupyter):

```python
from IPython.display import Image, display
display(Image(agent.get_graph(xray=True).draw_mermaid_png()))
```

## Functional API

Same agent, no `StateGraph`. `@task` marks a unit of work; `@entrypoint` is the top-level function. Tasks return futures — `.result()` blocks for the value.

```python
from langgraph.graph import add_messages
from langchain.messages import SystemMessage, HumanMessage, ToolCall
from langchain_core.messages import BaseMessage
from langgraph.func import entrypoint, task

@task
def call_llm(messages: list[BaseMessage]):
    return model_with_tools.invoke(
        [SystemMessage(content="You are a helpful assistant tasked with performing arithmetic on a set of inputs.")]
        + messages
    )

@task
def call_tool(tool_call: ToolCall):
    tool = tools_by_name[tool_call["name"]]
    return tool.invoke(tool_call)

@entrypoint()
def agent(messages: list[BaseMessage]):
    model_response = call_llm(messages).result()

    while True:
        if not model_response.tool_calls:
            break

        tool_result_futures = [call_tool(tc) for tc in model_response.tool_calls]
        tool_results = [fut.result() for fut in tool_result_futures]
        messages = add_messages(messages, [model_response, *tool_results])
        model_response = call_llm(messages).result()

    messages = add_messages(messages, model_response)
    return messages
```

**Stream updates:**

```python
messages = [HumanMessage(content="Add 3 and 4.")]
for chunk in agent.stream(messages, stream_mode="updates"):
    print(chunk)
    print("\n")
```

Note the parallel fan-out: dispatching `call_tool` for every tool call returns futures immediately, so multiple tools in one turn run concurrently.

⚠️ The `Annotated[list[AnyMessage], operator.add]` reducer matters — drop the annotation and each node return **replaces** `messages` instead of appending, and the loop loses history.

💡 Same agent, two surfaces: graph view exposes routing and parallelism for visualisation; functional view reads like normal Python with futures. Pick per use case — see [[LangGraph GS 04 - Choosing Graph vs Functional APIs]].
