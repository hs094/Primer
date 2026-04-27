# CO 02 — Application Structure

🔑 A LangGraph deployment is four files: `langgraph.json` (the manifest), the graph module, a dependency file, and an optional `.env` — everything else is project code.

## Required pieces

1. **`langgraph.json`** — declares dependencies, named graphs, and env file.
2. **Graph module(s)** — Python file(s) that build a compiled graph or a factory function returning one.
3. **Dependencies** — `pyproject.toml` or `requirements.txt`.
4. **`.env`** (optional) — local-only secrets; production uses the platform's env-var mechanism.

## Typical layout

```
my-app/
├── my_agent/
│   ├── utils/
│   │   ├── tools.py
│   │   ├── nodes.py
│   │   └── state.py
│   └── agent.py
├── .env
├── requirements.txt          # or pyproject.toml
└── langgraph.json
```

The package name (`my_agent/`) and module (`agent.py`) are referenced from `langgraph.json` — keep them stable, the deployment binds to the path.

## `langgraph.json`

```json
{
  "dependencies": ["langchain_openai", "./your_package"],
  "graphs": {
    "my_agent": "./your_package/your_file.py:agent"
  },
  "env": "./.env"
}
```

### Keys

- **`dependencies`** — list of pip-installable packages plus local paths (e.g. `./your_package`) that LangGraph should install. Local paths must be Python packages (have `pyproject.toml` or `setup.py`).
- **`graphs`** — map of `name → "path/to/file.py:variable"`. The variable must point to either a compiled `Pregel` graph or a callable that returns one. Each name becomes a deployable graph endpoint.
- **`env`** — relative path to a dotenv file, used in local dev. In production, set vars on the platform.
- **`dockerfile_lines`** — extra `RUN`/`COPY` lines injected into the deployment image (system libs, custom build steps).

For the full schema (including `python_version`, `pip_config_file`, `store`, `http`), see the LangGraph CLI configuration reference — it owns the canonical list.

## Graph module shape

The graph variable referenced from `langgraph.json` must exist at module top level:

```python
# your_package/your_file.py
from langgraph.graph import StateGraph, START, END

builder = StateGraph(State)
# ... add_node / add_edge ...
agent = builder.compile()    # <-- this is what `:agent` resolves to
```

A factory function works too:

```python
def make_agent():
    builder = StateGraph(State)
    ...
    return builder.compile()

agent = make_agent
```

## Dependencies

Use either:

```toml
# pyproject.toml
[project]
dependencies = ["langgraph", "langchain-anthropic", "httpx"]
```

or:

```
# requirements.txt
langgraph
langchain-anthropic
httpx
```

Keep `langgraph` itself pinned; the deployment image installs from this file.

## Env vars

Local development reads `.env` via the `env` key. Production deployments inject env vars at the platform layer — never commit secrets to `langgraph.json` or `.env`.

```python
import os
ANTHROPIC_API_KEY = os.environ["ANTHROPIC_API_KEY"]
```

⚠️ The path in `graphs` is resolved relative to `langgraph.json`. A typo there fails at deploy time, not import time — verify with `langgraph dev` locally.

## Multiple graphs

One project can expose several graphs side-by-side:

```json
{
  "graphs": {
    "research_agent": "./pkg/research.py:graph",
    "writer_agent": "./pkg/writer.py:graph"
  }
}
```

Each is independently invocable; they share dependencies and env but not state.

💡 Treat `langgraph.json` as the contract between your code and the runtime. Everything the deployment needs to find your graph lives there — keep paths stable, secrets out, and the dependency list minimal.
