# 05 — Tools

🔑 Tools are model-controlled functions; the LLM picks one from `tools/list` and the client invokes it via `tools/call`.

Source: https://modelcontextprotocol.io/docs/concepts/tools

## Methods
| Method | Purpose |
|---|---|
| `tools/list` | Discover tools (paginated) |
| `tools/call` | Invoke `{name, arguments}` |
| `notifications/tools/list_changed` | Server's tool set shifted |

## Tool Definition
- `name` (unique), `title`, `description`.
- `inputSchema` — JSON Schema for arguments (drives validation + LLM grounding).
- Optional `outputSchema` — validate `structuredContent`.
- Optional `annotations` — hints like `readOnlyHint`, `destructiveHint`. ⚠️ Treat untrusted unless server is trusted.

## Result Content
Returns a `content` array of typed blocks: `text`, `image`, `audio`, `resource_link`, `resource` (embedded). May also include `structuredContent` JSON when an `outputSchema` is declared.

## Error Modes
| Kind | Where |
|---|---|
| Protocol error (unknown tool, bad args) | JSON-RPC `error` |
| Tool execution failure (API down, bad input) | `result` with `isError: true` |

## FastMCP Example
```python
@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two integers."""
    return a + b
```
Type hints become the JSON Schema; the docstring becomes the description.

⚠️ Hosts should keep a human in the loop — show the call + args before executing destructive tools.

🧪 Try-it: `uv run mcp dev server.py` opens the Inspector to call tools by hand.

## Tags
[[MCP]] [[Tool Use]] [[FastMCP]] [[JSON-RPC]]
