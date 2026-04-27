# DP 01 — Run a Local Server

🔑 `langgraph dev` boots an in-memory Agent Server on `127.0.0.1:2024` with hot reload — no Docker, no Postgres, just edit and refresh. Studio auto-opens against the local URL.

## Install

Requires Python ≥ 3.11. The `[inmem]` extra pulls the dev-mode dependencies (in-memory checkpointer, no Docker).

```bash
pip install -U "langgraph-cli[inmem]"
# or
uv add "langgraph-cli[inmem]"
```

## Scaffold a project

```bash
langgraph new path/to/your/app --template new-langgraph-project-python
cd path/to/your/app
uv sync   # or: pip install -e .
```

Running `langgraph new` with no template opens an interactive picker.

## langgraph.json

```json
{
  "$schema": "https://langgra.ph/schema.json",
  "dependencies": ["."],
  "graphs": { "agent": "./src/agent.py:graph" },
  "env": ".env"
}
```

Full key surface:

| Key | Purpose |
|---|---|
| `dependencies` | `pyproject.toml`/`setup.py`/`requirements.txt` paths or PyPI names |
| `graphs` | Map of graph ID → `path/to/file.py:variable` (compiled graph or factory) |
| `env` | Path to `.env` *or* inline `{"KEY": "value"}` |
| `python_version` | `"3.11"` \| `"3.12"` \| `"3.13"` (default `3.11`); `node_version: 20` for JS |
| `pip_installer` | `"auto"` \| `"pip"` \| `"uv"` (v0.3+); `pip_config_file` for `pip.conf` |
| `dockerfile_lines` | Extra Dockerfile directives at build |
| `image_distro` | `"debian"` \| `"wolfi"` \| `"bookworm"` \| `"bullseye"` |
| `auth` | `{ "path": "./auth.py:auth", "openapi": {...} }` |
| `store` / `checkpointer` | Semantic-search index, backend, TTL, sweep interval |
| `http` | CORS, route disables (`disable_meta`, `disable_runs`, …) |
| `webhooks` | Outbound delivery (v0.5.36+); `api_version` pins server, e.g. `"0.2"` |
| `ui` | Named generative UI components |

## Environment

Drop secrets into `.env` next to `langgraph.json`:

```bash
LANGSMITH_API_KEY=lsv2_pt_...
OPENAI_API_KEY=sk-...
LANGSMITH_PROJECT=my-agent-dev   # optional override
```

Set `LANGSMITH_TRACING=false` to keep traces off the dashboard during local work.

## Launch the dev server

```bash
langgraph dev
```

Output:

```
- API:        http://127.0.0.1:2024
- Docs:       http://127.0.0.1:2024/docs
- Studio UI:  https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

Hot reload is on by default — saving `agent.py` reflects in the next request and in [[LangGraph DP 04 - LangSmith Studio|Studio]].

### Useful flags

| Flag | Default | Notes |
|---|---|---|
| `-c, --config` | `langgraph.json` | Alt config path |
| `--host` | `127.0.0.1` | Bind address |
| `--port` | `2024` | |
| `--no-reload` | — | Disable file-watch restart |
| `--n-jobs-per-worker` | `10` | Parallelism per worker |
| `--debug-port` | — | Attach a debugger |
| `--wait-for-client` | `False` | Block until debugger attaches |
| `--no-browser` | — | Don't auto-open Studio |
| `--studio-url` | `https://smith.langchain.com` | For self-hosted Studio |
| `--allow-blocking` | `False` | Permit sync I/O in nodes (v0.2.6+) |
| `--tunnel` | `False` | Public ngrok-style tunnel — Safari needs this since it blocks `localhost` cross-origin |

## Smoke-test

```python
from langgraph_sdk import get_client
import asyncio

client = get_client(url="http://localhost:2024")

async def main():
    async for chunk in client.runs.stream(
        None,                          # thread_id (None = stateless)
        "agent",                       # assistant_id from langgraph.json
        input={"messages": [{"role": "human", "content": "What is LangGraph?"}]},
    ):
        print(f"event={chunk.event} data={chunk.data}")

asyncio.run(main())
```

Sync variant: `get_sync_client(...)`. REST equivalent: `POST /runs/stream` with `assistant_id`, `input`, `stream_mode`.

## Caveats

- In-memory only — checkpoints vanish on restart. For persistence use [[LangGraph DP 02 - LangSmith Deployment|`langgraph up`]].
- One-process server, fine for development, not for load.
- Safari blocks `http://localhost` from `smith.langchain.com`; use `--tunnel`.

💡 `langgraph dev` is the inner-loop tool: edit `agent.py`, save, watch Studio re-render. Promote to `langgraph build` / `langgraph up` when you need Postgres-backed durability.
