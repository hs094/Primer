# TS 02 — Agent Chat UI

🔑 `langchain-ai/agent-chat-ui` is a Next.js chat front-end you point at any LangGraph deployment URL. Three knobs — graph ID, deployment URL, optional API key — and you have a shareable conversational interface with tool rendering, time travel, and generative UI baked in.

## Repo

[github.com/langchain-ai/agent-chat-ui](https://github.com/langchain-ai/agent-chat-ui) — Next.js + React, designed for graphs built with `create_agent` (and any graph that emits a `messages` channel).

What it renders out of the box:

- Streaming token-by-token responses.
- Tool calls + tool results as collapsible cards.
- Interrupted threads (auto-fetched and surfaced for resume).
- State forks and time travel (re-run from any step).
- Custom **generative UI** components defined under `langgraph.json` → `ui`.
- Configurable message hiding (system messages, internal tool chatter).

## Three ways to run it

### 1. Hosted — zero install

Visit `https://agentchat.vercel.app`, paste your deployment URL + graph ID, chat. Good for demos.

### 2. Scaffold a local copy

```bash
npx create-agent-chat-app --project-name my-chat-ui
cd my-chat-ui
pnpm install
pnpm dev
```

Opens on `http://localhost:3000`.

### 3. Clone the repo

```bash
git clone https://github.com/langchain-ai/agent-chat-ui.git
cd agent-chat-ui
pnpm install
pnpm dev
```

Pick this when you want to fork message rendering, swap themes, or wire bespoke generative UI components.

## Connect it to an agent

Three settings — entered in the UI on first load and persisted to localStorage, or pre-baked via `.env.local`:

| Setting | Source | Example |
|---|---|---|
| **Graph ID** | The key under `graphs` in `langgraph.json` | `agent` |
| **Deployment URL** | Local: [[LangGraph DP 01 - Run a Local Server\|`langgraph dev`]] prints it. Cloud: from LangSmith Deployments | `http://localhost:2024` or `https://my-agent-xxxx.us.langgraph.app` |
| **LangSmith API key** | Only required for hosted deployments — not for local agent servers | `lsv2_pt_...` |

`.env.local` for the cloned repo:

```bash
NEXT_PUBLIC_API_URL=http://localhost:2024
NEXT_PUBLIC_ASSISTANT_ID=agent
# Only needed when targeting a deployed graph behind LangSmith auth:
LANGSMITH_API_KEY=lsv2_pt_...
```

`NEXT_PUBLIC_*` vars get inlined into the client bundle; the `LANGSMITH_API_KEY` stays server-side and is proxied through a Next.js route.

## Local end-to-end loop

```bash
# Terminal 1 — the agent
cd my-agent
langgraph dev
# -> http://127.0.0.1:2024

# Terminal 2 — the UI
cd ../my-chat-ui
NEXT_PUBLIC_API_URL=http://127.0.0.1:2024 \
NEXT_PUBLIC_ASSISTANT_ID=agent \
pnpm dev
# -> http://localhost:3000
```

Edit `agent.py`, save, refresh the chat — hot reload on both sides.

## Hiding messages

The default behavior renders every message in the `messages` channel. To hide internal traffic (e.g. a planner's scratch chain), set the `do-not-render` content key per the repo's [Hiding Messages in the Chat](https://github.com/langchain-ai/agent-chat-ui#hiding-messages) guide. Useful when you have multi-agent fan-out where only the final reply belongs in the user view.

## Generative UI

Components declared under `ui` in `langgraph.json` can be emitted from a node and rendered by the chat as rich React widgets — forms, charts, custom approval dialogs.

```json
{
  "graphs": { "agent": "./src/agent.py:graph" },
  "ui": {
    "approval_card": "./src/ui/approval.tsx:ApprovalCard"
  }
}
```

A node emits a UI message with `name: "approval_card"` and a payload; the chat looks up the component, renders it, and any user interaction is routed back as a state update / `Command(resume=...)`. See the LangGraph generative UI guide on `docs.langchain.com` for the message envelope.

## Interrupts and resume

Any thread that hits an `interrupt(...)` shows up under the chat's "Interrupted" section. Clicking the thread reveals the interrupt payload and a resume affordance — same UX as [[LangGraph DP 04 - LangSmith Studio|Studio]] but inside the end-user surface. Use this for human-in-the-loop approvals where the operator isn't a developer.

## Customizing

Common forks: Tailwind theme + logo in `app/layout.tsx`; wrap pages in session middleware before exposing the deployment URL; override the `Message` component for reactions/badges; reuse the time-travel hook to surface a "raw state" toggle.

## Studio vs Agent Chat UI

| Surface | Audience | Strength |
|---|---|---|
| [[LangGraph DP 04 - LangSmith Studio\|Studio]] | builders | graph view, checkpoint surgery, hot reload |
| Agent Chat UI | end users / stakeholders | clean chat, tool cards, generative UI |

Both target the same deployment URL — pick by audience.

## Production checklist

- Set `NEXT_PUBLIC_API_URL` to the deployed graph's URL, not `localhost`.
- Proxy LangSmith auth through a Next.js route handler — never ship `LANGSMITH_API_KEY` to the browser.
- Configure CORS on the agent server (`http.cors.allow_origins` in `langgraph.json`, see [[LangGraph DP 02 - LangSmith Deployment|DP 02]]) to whitelist the chat origin.
- Hide internal/system messages before user demos.
- Pin the `langgraph-sdk` package the UI depends on so streaming envelopes don't drift from the server.

💡 Agent Chat UI is the fastest way to put a face on a graph. Treat it as a starter — fork early once you need brand or auth control.
