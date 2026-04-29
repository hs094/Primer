# 03 — Transports

🔑 Two official transports: **stdio** for local subprocesses and **Streamable HTTP** for remote servers — same JSON-RPC 2.0 frames either way.

Source: https://modelcontextprotocol.io/specification/2025-06-18/basic/transports

## stdio
- Host launches server as subprocess; talks over `stdin`/`stdout`.
- Messages are newline-delimited JSON; **no embedded newlines**.
- `stderr` is free for logs (host may capture or ignore).
- Best for local tools (filesystem, git, sqlite).

⚠️ Server **must not** print non-MCP chatter to stdout — it corrupts the stream.

## Streamable HTTP
- Single MCP endpoint (e.g. `https://example.com/mcp`) handling POST + GET.
- Client POSTs JSON-RPC; server replies either with `application/json` (one shot) or `text/event-stream` (SSE stream of multiple messages).
- Client may also GET to open a server-push SSE stream.
- Replaces the deprecated 2024-11-05 HTTP+SSE transport.

### Session Management
- Server may issue `Mcp-Session-Id` header on initialize; client echoes it on every later request.
- 404 on session ID → start a new `initialize`.
- Client DELETE on the endpoint cleanly ends a session.
- `MCP-Protocol-Version` header pins the negotiated version on each request.

## Choosing One
| Use | Pick |
|---|---|
| Local CLI/script integration | **stdio** |
| Remote SaaS server, multi-tenant | **Streamable HTTP** |
| Browser-hosted client | **Streamable HTTP** |

⚠️ **DNS-rebinding defense**: HTTP servers must validate `Origin`, bind to `127.0.0.1` for local use, and require auth.

💡 Websockets aren't a standard MCP transport — use Streamable HTTP for bidirectional needs.

## Tags
[[MCP]] [[JSON-RPC]] [[Transports]]
