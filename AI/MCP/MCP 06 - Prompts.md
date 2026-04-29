# 06 — Prompts

🔑 Prompts are user-controlled, parameterized message templates the server ships — typically surfaced as slash commands.

Source: https://modelcontextprotocol.io/docs/concepts/prompts

## Methods
| Method | Purpose |
|---|---|
| `prompts/list` | Discover prompts (paginated) |
| `prompts/get` | Render a prompt with arguments |
| `notifications/prompts/list_changed` | Set shifted |

## Prompt Shape
- `name`, optional `title`, `description`.
- `arguments`: list of `{name, description, required}` for templating.
- Response is `messages: [{role, content}]` — drop straight into a model call.

## Content Types in Messages
- `text` — most common.
- `image` / `audio` — base64 + `mimeType`.
- `resource` — embed a server resource directly into the conversation.

## FastMCP Example
```python
@mcp.prompt(title="Code Review")
def review_code(code: str) -> str:
    return f"Please review this code and call out issues:\n\n{code}"
```
A `str` return is wrapped as a single user message; return a list of `Message` objects for multi-turn templates.

## Where They Show Up
- Claude Desktop / VS Code: slash commands or a prompt picker.
- User picks → host calls `prompts/get` → prepends returned messages to the conversation.

💡 Use prompts for repeatable workflows ("summarize this PR", "draft release notes") — keeps boilerplate out of the user's chat.

⚠️ Validate arguments server-side; treat them as untrusted input even though the user picked the prompt.

## Tags
[[MCP]] [[Prompts]] [[FastMCP]]
