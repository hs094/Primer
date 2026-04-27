# DP 04 — LangSmith Studio

🔑 Studio is the visual front-end for any LangGraph server — local dev or deployed. Run `langgraph dev` and Studio auto-opens; paste a deployment URL and Studio attaches to prod. Graph view, threads, time travel, and interrupt UI in one place.

## What Studio gives you

- **Graph visualization** — live diagram of nodes/edges from `langgraph.json`.
- **Step-by-step replay** — every prompt, tool call, intermediate state.
- **Threads** — list of conversations with their checkpoints; re-run from any step.
- **Time travel** — jump to a past checkpoint, edit state, fork a new run.
- **Interrupt UX** — when a graph hits `interrupt(...)`, Studio shows the pending value and lets you `Command(resume=...)` from the UI.
- **Hot reload** — saves to your code reflect immediately when paired with [[LangGraph DP 01 - Run a Local Server|`langgraph dev`]].
- **Error capture** — exceptions land with their surrounding state for inspection.
- **Token + latency metrics** — sourced from [[LangGraph DP 03 - LangSmith Observability|LangSmith traces]].

## Connect to a local dev server

`langgraph dev` prints the Studio URL on startup:

```
- API:        http://127.0.0.1:2024
- Studio UI:  https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

The browser auto-opens (`--no-browser` to skip). Studio is hosted at `smith.langchain.com` but reads from your local server via `baseUrl=...`.

### Safari fix

Safari blocks `http://localhost` from `smith.langchain.com` by CORS policy. Use the public tunnel:

```bash
langgraph dev --tunnel
```

The CLI prints a `https://*.trycloudflare.com` URL to paste into Studio (`?baseUrl=...`).

## Connect to a deployment

For a [[LangGraph DP 02 - LangSmith Deployment|deployed]] graph, open the deployment in LangSmith and click **Studio**. The URL pattern is:

```
https://smith.langchain.com/studio/?baseUrl=<deployment-url>
```

Auth uses your LangSmith session — no extra API key required when going through the dashboard.

## Prerequisites

- LangSmith account (free signup at `smith.langchain.com`).
- `LANGSMITH_API_KEY` in `.env`.
- A `langgraph.json` pointing at compiled graphs.
- Python ≥ 3.11.

Minimal config Studio needs to render:

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent.py:graph"
  },
  "env": ".env"
}
```

## Threads + time travel

Each run is bound to a `thread_id`. Studio's **Threads** tab lists them with timestamps and message previews. Click a thread → see the checkpoint timeline. Click a checkpoint → "Fork from here" lets you:

1. Edit any field of the state.
2. Resume execution with the modified state.
3. Compare branches side-by-side.

This is the same primitive as `client.threads.update_state(thread_id, values, as_node=...)` from the SDK — Studio just gives you a form.

## Interrupts in the UI

When a node calls `interrupt({"question": "..."})`, the run pauses and Studio surfaces a panel with:

- The interrupt payload (rendered as JSON or a form).
- A **Resume** button that takes a value and resends as `Command(resume=value)`.
- The current state snapshot at the pause point.

Resume completes the run from where it paused — same semantics as a programmatic `client.runs.stream(thread_id, ..., command=Command(resume=...))`.

## Privacy mode

To use Studio without sending traces to LangSmith:

```bash
# .env
LANGSMITH_TRACING=false
LANGSMITH_API_KEY=lsv2_pt_...   # still needed for Studio auth
```

Studio still renders the graph and threads (it reads from the local server), but no run data uploads.

## Self-hosted Studio

Point `langgraph dev` at a self-hosted LangSmith instance:

```bash
langgraph dev --studio-url https://langsmith.internal.example.com
```

The CLI rewrites the printed Studio URL accordingly.

## When to reach for Studio

| Task | Studio | SDK / curl |
|---|---|---|
| Visualize node graph | yes | no |
| Inspect intermediate state | yes | tedious |
| Reproduce a bug from a thread | yes (fork from checkpoint) | possible via `update_state` |
| Drive a tight CI loop | no | yes ([[LangGraph TS 01 - Testing|pytest]]) |
| Demo to stakeholders | yes | no |

## Studio + Agent Chat UI

Studio is for builders — graph debugger, state surgeon. End-user-facing chat goes in [[LangGraph TS 02 - Agent Chat UI|Agent Chat UI]], which targets the same deployment URL but renders messages, not graph internals.

💡 If you're not in Studio while iterating on a graph, you're flying blind. `langgraph dev` opens it for free — keep the tab pinned.
