# 02 — Architecture

🔑 One **host** spawns one **client** per **server**; each client maintains a dedicated, stateful JSON-RPC connection.

Source: https://modelcontextprotocol.io/docs/learn/architecture

## Participants
| Role | Example | Job |
|---|---|---|
| **Host** | Claude Desktop, VS Code, Claude Code | AI app, manages clients |
| **Client** | One per server, lives inside host | Owns the connection |
| **Server** | Filesystem, Sentry, GitHub | Provides context + actions |

⚠️ "Server" doesn't imply remote — stdio servers run as local subprocesses.

## Capability Negotiation Handshake
1. Client → `initialize` with `protocolVersion`, `capabilities`, `clientInfo`.
2. Server → response with its own `capabilities`, `serverInfo`.
3. Client → `notifications/initialized` (fire-and-forget).
4. Normal request/response/notification traffic begins.

```json
{"jsonrpc":"2.0","id":1,"method":"initialize",
 "params":{"protocolVersion":"2025-06-18",
           "capabilities":{"sampling":{}},
           "clientInfo":{"name":"my-app","version":"1.0"}}}
```

## What Capabilities Declare
- Server: which primitives it offers (`tools`, `resources`, `prompts`) + whether each supports `listChanged` / `subscribe`.
- Client: whether it supports `sampling`, `elicitation`, `roots`, etc.

💡 Negotiation lets either side skip unsupported calls — no blind probing.

## Lifecycle
- **Initialize** → version + capabilities.
- **Operate** → primitive calls + notifications.
- **Shutdown** → transport-specific (close stdin, DELETE session, etc).

## Tags
[[MCP]] [[JSON-RPC]] [[Anthropic]]
