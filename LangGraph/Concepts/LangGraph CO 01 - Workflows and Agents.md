# CO 01 — Workflows and Agents

🔑 Workflows are LLM apps with predetermined control flow; agents let the LLM decide what to do next via tool calls — pick the simplest pattern that fits.

## The split

- **Workflow** — orchestration is hard-coded: nodes, edges, conditionals. Each step's role is fixed at design time.
- **Agent** — orchestration is delegated to the LLM: it loops, picks tools, decides when it's done. Tool definitions are the only constraint.

Start with a workflow; reach for an agent only when the path is too branchy or unknown to encode by hand.

## Augmenting the LLM

Three primitives sit underneath every pattern:

- `llm.bind_tools(tools)` — make functions callable from a model turn.
- `llm.with_structured_output(Schema)` — Pydantic-validated outputs.
- A checkpointer for memory (see [[LangGraph CO 05 - Persistence (Checkpointers)]]).

## Prompt chaining

Sequential LLM calls; each consumes the prior output. Conditional gate can skip refinement when the first draft passes.

```python
workflow = StateGraph(State)
workflow.add_node("generate_joke", generate_joke)
workflow.add_node("improve_joke", improve_joke)
workflow.add_node("polish_joke", polish_joke)
workflow.add_edge(START, "generate_joke")
workflow.add_conditional_edges(
    "generate_joke", check_punchline, {"Fail": "improve_joke", "Pass": END}
)
workflow.add_edge("improve_joke", "polish_joke")
workflow.add_edge("polish_joke", END)
chain = workflow.compile()
```

## Parallelization

Fan multiple independent calls out from `START`, then aggregate. Used for speed (different subtasks) or confidence (same task, multiple votes).

```python
parallel_builder.add_edge(START, "call_llm_1")
parallel_builder.add_edge(START, "call_llm_2")
parallel_builder.add_edge(START, "call_llm_3")
parallel_builder.add_edge("call_llm_1", "aggregator")
parallel_builder.add_edge("call_llm_2", "aggregator")
parallel_builder.add_edge("call_llm_3", "aggregator")
```

## Routing

A classifier node picks the downstream branch — typed via `with_structured_output`.

```python
class Route(BaseModel):
    step: Literal["poem", "story", "joke"] = Field(...)

router = llm.with_structured_output(Route)

router_builder.add_conditional_edges(
    "llm_call_router",
    route_decision,
    {"llm_call_1": "llm_call_1", "llm_call_2": "llm_call_2", "llm_call_3": "llm_call_3"},
)
```

## Orchestrator–worker

One LLM plans subtasks; workers run in parallel; a synthesizer merges. Worker count is dynamic — use the `Send` API.

```python
from langgraph.types import Send

def assign_workers(state: State):
    return [Send("llm_call", {"section": s}) for s in state["sections"]]

orchestrator_worker_builder.add_conditional_edges(
    "orchestrator", assign_workers, ["llm_call"]
)
```

Workers report into a shared reducer key:

```python
class State(TypedDict):
    completed_sections: Annotated[list, operator.add]
```

## Evaluator–optimizer

Generator drafts; evaluator grades; loop until accepted. Common for translation, code, refinement.

```python
class Feedback(BaseModel):
    grade: Literal["funny", "not funny"] = Field(...)
    feedback: str = Field(...)

evaluator = llm.with_structured_output(Feedback)

optimizer_builder.add_conditional_edges(
    "llm_call_evaluator",
    route_joke,
    {"Accepted": END, "Rejected + Feedback": "llm_call_generator"},
)
```

## The agent loop

LLM with tools → tool node → back to LLM, until the LLM stops emitting tool calls.

```python
llm_with_tools = llm.bind_tools(tools)

def llm_call(state: MessagesState):
    return {"messages": [llm_with_tools.invoke([SystemMessage(...)] + state["messages"])]}

def tool_node(state: dict):
    result = []
    for tool_call in state["messages"][-1].tool_calls:
        tool = tools_by_name[tool_call["name"]]
        observation = tool.invoke(tool_call["args"])
        result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
    return {"messages": result}

def should_continue(state: MessagesState) -> Literal["tool_node", END]:
    if state["messages"][-1].tool_calls:
        return "tool_node"
    return END

agent_builder = StateGraph(MessagesState)
agent_builder.add_node("llm_call", llm_call)
agent_builder.add_node("tool_node", tool_node)
agent_builder.add_edge(START, "llm_call")
agent_builder.add_conditional_edges("llm_call", should_continue, ["tool_node", END])
agent_builder.add_edge("tool_node", "llm_call")
agent = agent_builder.compile()
```

## Graph API vs Functional API

Both compile down to the same runtime ([[LangGraph CO 03 - Pregel Runtime]]). Graph API (`StateGraph`) is explicit nodes/edges; Functional API (`@task`, `@entrypoint`) wraps Python functions and infers structure.

⚠️ Hard-coded paths beat agentic loops when correctness matters and the branches are knowable. Don't reach for an agent because it sounds cool.

💡 Workflow patterns scale linearly with predictability; agents trade predictability for flexibility. Compose: an agent inside one node of a workflow is often the right shape.
