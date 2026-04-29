# MCP — INDEX

🔑 Model Context Protocol: open standard for wiring LLMs to tools, data, and prompts. Read in order.

Source: https://modelcontextprotocol.io/

## Notes
1. [[MCP 01 - Overview]] — what MCP is, who it's for, the three primitives.
2. [[MCP 02 - Architecture]] — host / client / server, capability negotiation, lifecycle.
3. [[MCP 03 - Transports]] — stdio vs Streamable HTTP, sessions, when to use each.
4. [[MCP 04 - Resources]] — read-only context data; `resources/list` + `resources/read`.
5. [[MCP 05 - Tools]] — model-controlled actions; JSON Schema args, structured output.
6. [[MCP 06 - Prompts]] — user-picked templates; slash-command UX.
7. [[MCP 07 - Sampling]] — server-initiated LLM completions via the client's model.
8. [[MCP 08 - Server SDK Python]] — `mcp` PyPI + FastMCP decorators.
9. [[MCP 09 - Client SDK]] — `ClientSession` + Claude tool-use bridge.
10. [[MCP 10 - Security and Best Practices]] — confused deputy, token passthrough, SSRF, session hijack.

## Mental Model
| Direction | Server primitives | Client primitives |
|---|---|---|
| Server → Client offers | tools, resources, prompts | — |
| Client → Server offers | — | sampling, elicitation, logging |

## Markers in Notes
- 🔑 core — the one-line takeaway.
- ⚠️ gotcha — easy to get wrong.
- 💡 tip — pragmatic guidance.
- 🧪 try-it — runnable next step.

## Tags
[[MCP]] [[Anthropic]] [[Tool Use]] [[JSON-RPC]] [[FastMCP]]
