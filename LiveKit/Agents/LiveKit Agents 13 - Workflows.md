# 13 — Workflows

🔑 Compose voice apps from four primitives: `AgentSession` (orchestrator), `Agent` (long-lived control), tasks (short-lived typed results), and task groups (ordered, revisitable steps).

Source: https://docs.livekit.io/agents/build/workflows/

## The four constructs
- **`AgentSession`** — single orchestrator owning the audio pipeline; can host multiple agents over its lifetime.
- **`Agent`** — holds long-lived control of the session. Swap when a different reasoning style or tool set is needed.
- **Tasks** — short-lived, run-to-completion, typed return value. Don't persist after returning.
- **Task groups** — sequential multi-step flows where earlier steps can be revisited (e.g. correcting a previously captured field).

## Design heuristics
- New `Agent` per distinct reasoning behavior or tool surface.
- Task per discrete operation (consent capture, ID verification).
- Task group when ordered steps may need correction.
- Map conversation phases before writing code.
- Build incrementally with tests; announce handoffs and choose whether to preserve `chat_ctx` per transition.

## Skeleton

```python
class IntakeAgent(Agent):
    @function_tool()
    async def begin_diagnosis(self, context: RunContext) -> Agent:
        return DiagnosisAgent(chat_ctx=self.chat_ctx.copy(exclude_instructions=True))

class DiagnosisAgent(Agent):
    ...
```

The session stays put; only the active `Agent` swaps. Tools execute side effects and trigger handoffs by returning a new `Agent`.

## Tags
[[LiveKit]] [[Agents]] [[Workflows]] [[AgentSession]] [[Tasks]]
