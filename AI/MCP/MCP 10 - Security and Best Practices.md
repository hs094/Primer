# 10 — Security and Best Practices

🔑 MCP gives models real-world reach — sandbox the server, scope the tokens, and keep a human in the loop on anything destructive.

Source: https://modelcontextprotocol.io/specification/2025-06-18/basic/security_best_practices

## Top Attack Vectors
| Attack | Mitigation |
|---|---|
| **Confused deputy** (proxy reuses static client_id, attacker steals consent cookie) | Per-client consent UI **before** redirecting to upstream IdP; validate `redirect_uri` exactly |
| **Token passthrough** (server forwards client tokens to upstream APIs) | **Forbidden**. Only accept tokens issued *to this MCP server* |
| **SSRF** via OAuth metadata URLs | Enforce HTTPS; block `127.0.0.1`, `169.254.169.254`, `10/8`, `172.16/12`, `192.168/16` |
| **Session hijacking** | Cryptographically random session IDs; bind to user_id; never use sessions for authn |
| **DNS rebinding** on local HTTP servers | Validate `Origin`; bind to `127.0.0.1`; require auth |
| **Local server compromise** (malicious startup command) | Show full command pre-consent; sandbox; warn on `sudo`/`rm -rf` |

## Server Hygiene
- Validate every input (especially URIs and tool args).
- Rate-limit tool calls.
- Sanitize outputs before they reach the model.
- Log tool usage with correlation IDs for audit.
- **Scope minimization**: ship a baseline scope (`mcp:tools-basic`); elevate via `WWW-Authenticate` challenges.

## Client/Host Hygiene
- Show tools + arguments to the user before invoking destructive ones.
- Treat tool annotations as untrusted unless the server is trusted.
- Validate tool results against `outputSchema` before passing to LLM.
- Implement timeouts on every call.

## OAuth for HTTP Servers
- Use OAuth 2.1 with PKCE for remote servers.
- Single-use cryptographic `state` parameter, set **after** consent approval.
- `__Host-` cookie prefix, `Secure`, `HttpOnly`, `SameSite=Lax`.

⚠️ A "pure proxy" mindset rots into a security hole — always validate token audience as if you'll add controls tomorrow.

## Tags
[[MCP]] [[Security]] [[OAuth]] [[Anthropic]]
