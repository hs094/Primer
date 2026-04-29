# 07 — Sampling

🔑 Sampling inverts the usual flow: the **server** asks the **client** to run an LLM completion — the server stays model-agnostic and needs no API key.

Source: https://modelcontextprotocol.io/docs/concepts/sampling

## Why It Exists
- Lets server-side agentic logic call a model without bundling an SDK or holding credentials.
- Client owns model choice, billing, and user approval.

## Method
- `sampling/createMessage` (server → client).
- Client must declare `"sampling": {}` capability during `initialize`.

## Request Fields
| Field | Use |
|---|---|
| `messages` | Conversation so far (text/image/audio) |
| `systemPrompt` | Optional system instruction |
| `maxTokens` | Cap |
| `modelPreferences` | Hints + priorities (see below) |
| `includeContext` | Pull in server's own context |

### Model Preferences
- `hints: [{name: "claude-3-sonnet"}, ...]` — substring match, may map across providers.
- `costPriority`, `speedPriority`, `intelligencePriority` (each 0–1).
- Advisory; client picks the actual model.

## Response
```json
{"role":"assistant",
 "content":{"type":"text","text":"…"},
 "model":"claude-3-sonnet-20240307",
 "stopReason":"endTurn"}
```

## Human-in-the-Loop
⚠️ Spec **strongly** recommends the client show the prompt + draft response to the user for approval — sampling lets servers spend the user's tokens.

💡 Use sampling when a tool needs sub-reasoning (e.g. "summarize this doc before returning") without coupling to a specific model.

## Tags
[[MCP]] [[Sampling]] [[Tool Use]]
