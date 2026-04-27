# PT 02 — Interrupts (Human-in-the-Loop)

🔑 `interrupt()` pauses a graph mid-node, persists state via the checkpointer, and waits indefinitely until the caller resumes with `Command(resume=...)` on the same `thread_id`.

## Requirements

Three things must be in place:

1. A checkpointer (memory or durable — see [[LangGraph CO 05 - Persistence (Checkpointers)]]).
2. A `thread_id` in `config={"configurable": {"thread_id": ...}}`.
3. A JSON-serialisable payload passed to `interrupt(...)`.

## Basic Pause / Resume

```python
from langgraph.types import interrupt, Command

def approval_node(state: State):
    approved = interrupt("Do you approve this action?")
    return {"approved": approved}

config = {"configurable": {"thread_id": "thread-1"}}
result = graph.invoke({"input": "data"}, config=config, version="v2")

# Resume — same thread_id, same graph instance
graph.invoke(Command(resume=True), config=config, version="v2")
```

The value passed to `interrupt(...)` becomes the chunk surfaced to the caller; the value passed to `Command(resume=...)` becomes the return value of the `interrupt()` call inside the node.

## Approve / Reject Routing

Combine `interrupt()` with `Command(goto=...)` to branch on the human decision:

```python
from typing import Literal, Optional, TypedDict
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt

class ApprovalState(TypedDict):
    action_details: str
    status: Optional[Literal["pending", "approved", "rejected"]]

def approval_node(state: ApprovalState) -> Command[Literal["proceed", "cancel"]]:
    decision = interrupt({
        "question": "Approve this action?",
        "details": state["action_details"],
    })
    return Command(goto="proceed" if decision else "cancel")

def proceed_node(state): return {"status": "approved"}
def cancel_node(state): return {"status": "rejected"}

builder = StateGraph(ApprovalState)
builder.add_node("approval", approval_node)
builder.add_node("proceed", proceed_node)
builder.add_node("cancel", cancel_node)
builder.add_edge(START, "approval")
builder.add_edge("proceed", END)
builder.add_edge("cancel", END)

graph = builder.compile(checkpointer=MemorySaver())

config = {"configurable": {"thread_id": "approval-123"}}
graph.invoke({"action_details": "Transfer $500", "status": "pending"}, config=config)
graph.invoke(Command(resume=True), config=config)
```

## Edit / Review Pattern

Surface current content, accept the edited version as the resume value, write it back to state:

```python
def review_node(state: ReviewState):
    updated = interrupt({
        "instruction": "Review and edit this content",
        "content": state["generated_text"],
    })
    return {"generated_text": updated}
```

## Tool-Call Approval

Drop `interrupt()` directly inside a tool to gate execution. The resume payload can also carry edits to override the original tool arguments:

```python
from langchain.tools import tool

@tool
def send_email(to: str, subject: str, body: str):
    """Send an email to a recipient."""
    response = interrupt({
        "action": "send_email",
        "to": to, "subject": subject, "body": body,
        "message": "Approve sending this email?",
    })
    if response.get("action") == "approve":
        final_to = response.get("to", to)
        final_subject = response.get("subject", subject)
        final_body = response.get("body", body)
        return f"Email sent to {final_to} with subject '{final_subject}'"
    return "Email cancelled by user"
```

## Validation Loop

A single node can call `interrupt()` repeatedly to re-prompt on bad input — each iteration creates a fresh pause:

```python
def get_age_node(state):
    prompt = "What is your age?"
    while True:
        answer = interrupt(prompt)
        if isinstance(answer, int) and answer > 0:
            return {"age": answer}
        prompt = f"'{answer}' is not a valid age. Please enter a positive number."
```

## Multiple Parallel Interrupts

When parallel branches both call `interrupt()`, the first invoke yields a list under `__interrupt__`. Resume them all at once with a dict mapping `interrupt.id` to the resume value:

```python
interrupted = graph.invoke({"vals": []}, config)

resume_map = {
    i.id: f"answer for {i.value}"
    for i in interrupted["__interrupt__"]
}
result = graph.invoke(Command(resume=resume_map), config)
```

## Static Interrupts (Breakpoints)

Pause before/after specific nodes without calling `interrupt()`. Resume by passing `None` as the input:

```python
graph = builder.compile(
    interrupt_before=["node_a"],
    interrupt_after=["node_b", "node_c"],
    checkpointer=checkpointer,
)

graph.invoke(inputs, config=config)
graph.invoke(None, config=config)  # resume
```

`interrupt_before` / `interrupt_after` can also be passed at runtime to `invoke()` / `stream()`.

## Streaming Alongside Interrupts

In a streaming UI, watch `updates` for the `__interrupt__` key, render the prompt, and resume:

```python
async for chunk in graph.astream(
    initial_input,
    stream_mode=["messages", "updates"],
    subgraphs=True,
    config=config,
    version="v2",
):
    if chunk["type"] == "updates" and "__interrupt__" in chunk["data"]:
        info = chunk["data"]["__interrupt__"][0].value
        user_response = get_user_input(info)
        initial_input = Command(resume=user_response)
        break
```

## Rules That Bite

- **Never wrap `interrupt()` in a bare `try/except`.** It pauses by raising a special exception; catching `Exception` swallows the pause. Wrap unrelated work separately or catch only specific exception classes.
- **Don't reorder, conditionally skip, or loop dynamically over `interrupt()` calls** within a node — resumes match by call index, so the order must be stable across runs.
- **Payloads must be JSON-serialisable.** No functions, no class instances. Use plain dicts / primitives.
- **Side effects before `interrupt()` must be idempotent.** The node re-runs from the top on resume — `db.upsert(...)` is fine, `db.append_to_history(...)` will duplicate. Move non-idempotent work after the interrupt or into a separate node.

💡 `interrupt()` + `Command(resume=...)` covers approve/reject, edit/review, validation loops, and tool-call gating — but the node body must be replay-safe, the payload JSON-serialisable, and the interrupt order stable.
