# 08 — Server SDK (Python)

🔑 The official `mcp` PyPI package ships **FastMCP**, a decorator-driven API that turns plain Python into a spec-compliant MCP server.

Source: https://github.com/modelcontextprotocol/python-sdk

## Install
```bash
uv add "mcp[cli]"
```

## Minimal Server
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two ints."""
    return a + b

@mcp.resource("greeting://{name}")
def greet(name: str) -> str:
    return f"Hello, {name}!"

@mcp.prompt(title="Code Review")
def review(code: str) -> str:
    return f"Review:\n\n{code}"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

## Decorators
| Decorator | Becomes |
|---|---|
| `@mcp.tool()` | Listed by `tools/list`, called via `tools/call` |
| `@mcp.resource("uri://{x}")` | Listed by `resources/list`, fetched via `resources/read` |
| `@mcp.prompt()` | Listed by `prompts/list`, rendered via `prompts/get` |

💡 Type hints generate the JSON Schema; Pydantic models / TypedDicts / dataclasses drive structured output validation automatically.

## Transports
```python
mcp.run(transport="stdio")          # local subprocess
mcp.run(transport="streamable-http") # remote, prod-ready
mcp.run(transport="sse")             # legacy
```

## Dev Loop
- `uv run mcp dev server.py` — boots the server + opens the **MCP Inspector** to poke tools/resources/prompts by hand.
- `mcp install server.py` — registers it with Claude Desktop.

⚠️ For stdio servers, never `print()` to stdout — use `logging` (goes to stderr) or you'll corrupt the JSON-RPC stream.

## Tags
[[MCP]] [[FastMCP]] [[Tool Use]]
