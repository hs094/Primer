# 04 — Resources

🔑 Resources are read-only data the server exposes by URI — files, schemas, records — for the host to splice into model context.

Source: https://modelcontextprotocol.io/docs/concepts/resources

## Methods
| Method | Purpose |
|---|---|
| `resources/list` | Enumerate available resources (paginated) |
| `resources/read` | Fetch contents by URI |
| `resources/templates/list` | List parameterized URI templates (RFC 6570) |
| `resources/subscribe` | Watch one URI for updates |

## Shape of a Resource
- `uri` (unique), `name`, optional `title`, `description`, `mimeType`, `size`.
- Contents are either `text` or base64 `blob`.
- Optional **annotations**: `audience` (`user`/`assistant`), `priority` 0–1, `lastModified`.

## URI Schemes
- `file://` — filesystem-like (need not be real FS).
- `https://` — only when the client can fetch directly without the server.
- `git://`, custom schemes welcome (RFC 3986).

## Notifications
- `notifications/resources/list_changed` — list shifted; client should re-list.
- `notifications/resources/updated` — subscribed URI changed; re-read.

## FastMCP Example
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("docs")

@mcp.resource("doc://{slug}")
def get_doc(slug: str) -> str:
    return (DOCS / f"{slug}.md").read_text()
```

⚠️ User-driven, not model-driven: hosts typically show a picker; the model doesn't auto-select resources the way it auto-selects tools.

💡 Use `priority` annotations to nudge hosts toward the important context first.

## Tags
[[MCP]] [[Resources]] [[FastMCP]]
