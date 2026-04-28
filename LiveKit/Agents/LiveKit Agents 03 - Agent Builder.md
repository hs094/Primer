# 03 — Agent Builder

🔑 Browser-based no-code prototyper for simple voice agents — emits production Python you can later extend in the SDK.

Source: https://docs.livekit.io/agents/start/builder/

## What you configure
- **Agent name** — used for dispatch routing
- **Instructions** — system prompt with dynamic variables
- **Greeting** — optional opening utterance
- **Models** — any LiveKit Inference STT, LLM, TTS combination
- **Tools** — three flavors:
  - HTTP API endpoints
  - Client-side RPC methods (frontend-executed)
  - MCP servers

## Advanced
- **Job metadata** parsed as JSON into prompt variables
- **Secrets** for API keys / credentials
- **End-of-call summaries** posted via HTTP webhook
- Background voice cancellation and preemptive generation enabled by default

## Workflow
1. Edit prompt + config in the UI; live preview tests it immediately
2. **Deploy agent** spins up production instances
3. Monitor sessions in **Agent insights**
4. **Convert to code** when you outgrow the builder — exports a Python SDK project

## Limitations (require dropping to SDK)
Workflows, agent handoffs, tasks, virtual avatars, vision, realtime models, the testing/eval harness.

## Billing
No extra charge — counts against standard LiveKit Cloud quotas + inference credits.

## Tags
[[LiveKit]] [[Agents]] [[NoCode]] [[Prototyping]] [[MCP]]
