# 09 — Client SDK

🔑 `ClientSession` from the `mcp` package owns one connection — initialize, list, call — then plug the tools into Claude's tool-use API.

Source: https://modelcontextprotocol.io/docs/develop/build-client

## Connect (stdio)
```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

params = StdioServerParameters(command="python", args=["server.py"])

async with stdio_client(params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        tools = (await session.list_tools()).tools
        result = await session.call_tool("add", {"a": 2, "b": 3})
```

## Bridge to Claude API Tool Use
```python
from anthropic import Anthropic

anthropic = Anthropic()
available = [{"name": t.name,
              "description": t.description,
              "input_schema": t.inputSchema} for t in tools]

resp = anthropic.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": query}],
    tools=available,
)

for block in resp.content:
    if block.type == "tool_use":
        result = await session.call_tool(block.name, block.input)
        # feed `result.content` back as a tool_result block, then re-call Claude
```

## Streamable HTTP
Swap `stdio_client(params)` for `streamablehttp_client(url, headers={...})` — same `ClientSession` interface.

## Capability Discovery
- `session.list_tools()`, `list_resources()`, `list_prompts()`, `list_resource_templates()`.
- Subscribe handlers for `notifications/*/list_changed` to refresh the registry on the fly.

💡 Use `AsyncExitStack` to manage lifetimes when juggling multiple servers in one host.

⚠️ Each MCP server gets its own `ClientSession` — never multiplex servers on one session.

## Tags
[[MCP]] [[Tool Use]] [[Anthropic]] [[FastMCP]]
