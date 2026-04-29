# 01 — Overview

🔑 MCP is an open standard from Anthropic that gives LLMs a USB-C-style port to plug into external tools, data, and prompts.

Source: https://modelcontextprotocol.io/

## What Problem It Solves
- Every AI app used to ship a bespoke integration per data source / tool.
- MCP standardizes that wiring so any compliant client (Claude Desktop, ChatGPT, Cursor, VS Code) talks to any compliant server.
- Build one server, integrate everywhere.

## Core Shape
| Layer | Role |
|---|---|
| **Data layer** | JSON-RPC 2.0 messages — lifecycle, primitives, notifications |
| **Transport layer** | stdio or Streamable HTTP — frames the bytes |

## Three Server Primitives
- **Tools** — callable functions the model invokes (`tools/call`).
- **Resources** — read-only context data exposed by URI (`resources/read`).
- **Prompts** — reusable templates the user picks (`prompts/get`).

💡 Think: tools = verbs, resources = nouns, prompts = recipes.

## Three Client Primitives
- **Sampling** — server asks client's LLM to complete (no server API key needed).
- **Elicitation** — server asks the user a follow-up question.
- **Logging** — server emits log lines to client.

## Why It Matters
- Devs ship integrations once.
- Hosts gain a growing ecosystem (filesystem, Sentry, GitHub, Notion, Blender, …).
- Users keep control: connections are explicit and inspectable.

## Tags
[[MCP]] [[Anthropic]] [[JSON-RPC]] [[Tool Use]]
