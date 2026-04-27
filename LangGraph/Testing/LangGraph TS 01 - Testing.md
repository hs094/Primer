# TS 01 — Testing

🔑 Two layers: pure pytest unit tests against compiled graphs (with `MemorySaver`) for fast inner-loop checks, and `@pytest.mark.langsmith` integration tests that log inputs/outputs/feedback to a LangSmith project for regression tracking + dataset curation.

## Layer 1 — local pytest

Build the graph fresh per test, compile with a new `MemorySaver`, exercise via `.invoke` / `.astream`. A new saver per test isolates checkpoints — no leaked thread state across cases.

```python
import pytest
from langgraph.checkpoint.memory import MemorySaver

@pytest.fixture
def compiled_graph():
    return build_my_graph().compile(checkpointer=MemorySaver())

def test_basic_flow(compiled_graph):
    result = compiled_graph.invoke(
        {"my_key": "initial_value"},
        config={"configurable": {"thread_id": "1"}},
    )
    assert result["my_key"] == "expected_value"
```

### Test a single node

Compiled graphs expose nodes by name — skip orchestration when only node logic matters:

```python
out = compiled_graph.nodes["node1"].invoke({"my_key": "initial_value"})
```

### Drive partial execution

Two knobs exercise sub-graphs without restructuring:

- `interrupt_after=["nodeA"]` on `.invoke` — stop after `nodeA` runs.
- `update_state(config, values, as_node="nodeA")` — push state in *as if* `nodeA` emitted it, then resume.

### Mock the LLM

Use LangChain fakes so tests don't hit a real model. Inject via your graph factory — don't hard-code model construction inside nodes.

```python
from langchain_core.language_models.fake_chat_models import FakeListChatModel

llm = FakeListChatModel(responses=['{"action":"search"}', '{"action":"respond"}'])
graph = build_my_graph(llm=llm).compile(checkpointer=MemorySaver())
```

`FakeListLLM` covers non-chat models.

### Snapshot state + async

`get_state(config).values` returns the StateSnapshot dict — diff via `syrupy` / `pytest-snapshot`. For streaming, pair `pytest-asyncio` with `astream`:

```python
@pytest.mark.asyncio
async def test_streaming(compiled_graph):
    chunks = [ev async for ev in compiled_graph.astream(
        {"my_key": "x"}, config={"configurable": {"thread_id": "1"}})]
    assert chunks
```

## Layer 2 — `@pytest.mark.langsmith`

For regression suites you want tracked over time, decorate tests so each run logs to a LangSmith project. Inputs, outputs, and feedback become rows in a dataset you can chart.

```python
import pytest
from langsmith import testing as t

@pytest.mark.langsmith
def test_sql_generation_select_all() -> None:
    user_query = "Get all users from the customers table"
    t.log_inputs({"user_query": user_query})

    sql = generate_sql(user_query)
    t.log_outputs({"sql": sql})
    t.log_reference_outputs({"sql": "SELECT * FROM customers"})

    t.log_feedback(key="valid_sql", score=is_valid_sql(sql))
    assert "SELECT" in sql.upper()
```

`langsmith.testing` helpers: `t.log_inputs({...})`, `t.log_outputs({...})`, `t.log_reference_outputs({...})`, `t.log_feedback(key=, score=)`, `with t.trace_feedback():` (isolates evaluator traces). Test arguments auto-log as inputs; mark reference outputs via `output_keys=[...]` on the decorator.

Combine with `@pytest.mark.parametrize` for table tests and `pytest-asyncio` for async. Run via:

```bash
LANGSMITH_TEST_SUITE='SQL app tests' pytest tests/
LANGSMITH_TEST_CACHE=tests/cassettes pytest tests/   # cache LLM calls
LANGSMITH_TEST_TRACKING=false pytest tests/          # dry-run, skip upload
pytest --langsmith-output tests/                     # rich terminal table
```

`LANGSMITH_TEST_CACHE` records LLM responses to disk so reruns are deterministic and free.

## Layer 3 — `evaluate()` over datasets

For broader sweeps, curate a LangSmith dataset and run `evaluate()`:

```python
from langsmith import evaluate

def target(inputs: dict) -> dict:
    return {"answer": agent.invoke(inputs)["messages"][-1].content}

def correctness(run, example) -> dict:
    return {"key": "correct", "score": run.outputs["answer"] == example.outputs["answer"]}

evaluate(target, data="my-dataset", evaluators=[correctness], experiment_prefix="agent-v1.2")
```

Curate datasets from production [[LangGraph DP 03 - LangSmith Observability|traces]] (right-click → "Add to dataset") so the eval set tracks reality.

## Choosing a layer

| Need | Layer |
|---|---|
| Catch regressions in a node's logic | 1 (pytest + MemorySaver) |
| Verify state transitions step-by-step | 1 (`interrupt_after` / `update_state`) |
| Track quality drift across versions | 2 or 3 (`@pytest.mark.langsmith`, `evaluate`) |
| Compare two prompts on the same dataset | 3 (`evaluate` with `experiment_prefix`) |

💡 Layer 1 keeps CI green. Layer 2/3 keeps the *agent* good. You need both — pure unit tests can't tell you the model regressed; eval suites can't tell you a tool function broke.
