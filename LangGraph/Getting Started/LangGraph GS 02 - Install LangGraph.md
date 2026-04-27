# GS 02 — Install LangGraph

🔑 `pip install -U langgraph` (or `uv add langgraph`) on Python 3.10+ — install LangChain and provider packages separately for actual model access.

## Base install

**pip:**

```bash
pip install -U langgraph
```

**uv:**

```bash
uv add langgraph
```

Requires **Python 3.10+**.

The `langgraph` package gives you the runtime: `StateGraph`, `START`, `END`, `MessagesState`, the functional API (`@entrypoint`, `@task`), checkpointers, `interrupt`, `Command`, etc. It does **not** include any LLM provider.

## LangChain (typical companion)

LangGraph nodes usually call an LLM and bind tools. The docs use LangChain for both:

```bash
pip install -U langchain
```

or

```bash
uv add langchain
```

This gives you `init_chat_model`, `langchain.tools.tool`, `langchain.messages` (`SystemMessage`, `HumanMessage`, `AIMessage`, `ToolMessage`, `AnyMessage`), and `with_structured_output`. You're not required to use LangChain — any SDK that returns messages works — but every example in the docs assumes it.

## LLM provider packages

Each provider ships in its own package; install only what you use. For the quickstart's Claude examples:

```bash
pip install -U langchain-anthropic
```

Then set the key in your environment:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

OpenAI:

```bash
pip install -U langchain-openai
```

```bash
export OPENAI_API_KEY=sk-...
```

See the LangChain integrations index for the full list.

## Verify the install

```python
import langgraph
from langgraph.graph import StateGraph, START, END, MessagesState

print(langgraph.__version__)
```

Smoke-test with the hello-world graph from [[LangGraph GS 01 - Overview]] — no API key needed, since the node is a stub.

## Optional extras

- **`langgraph-checkpoint-sqlite`** / **`langgraph-checkpoint-postgres`** — persistent checkpointers for durable runs. The in-memory `MemorySaver` ships with the base package and is fine for local dev; swap in SQLite or Postgres for anything you want to survive a process restart.
- **LangSmith** — set `LANGSMITH_API_KEY` and `LANGSMITH_TRACING=true` in your environment to get traces of every node execution. No code change required.

```bash
export LANGSMITH_API_KEY=lsv2_...
export LANGSMITH_TRACING=true
```

## Recommended layout (uv)

```bash
uv init my-agent
cd my-agent
uv add langgraph langchain langchain-anthropic
uv run python -c "from langgraph.graph import StateGraph; print('ok')"
```

Pin versions in `pyproject.toml` once you're past prototyping — LangGraph and LangChain both move fast, and minor releases occasionally rename or relocate symbols.

⚠️ If imports fail with `cannot import name 'StateGraph'` or similar, you're almost always pinned to an old `langgraph` version. Check `pip show langgraph`; the public API stabilised mid-2024 and the layout has changed since.

💡 Treat `langgraph` + `langchain` + one provider package as the minimum baseline. Everything else (checkpointers, integrations, CLI) is opt-in. Next: [[LangGraph GS 03 - Quickstart]].
