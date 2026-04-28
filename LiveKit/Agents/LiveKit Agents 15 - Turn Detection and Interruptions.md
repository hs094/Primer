# 15 — Turn Detection and Interruptions

🔑 Pick one of five turn strategies (turn-detector model, realtime model, VAD, STT endpointing, manual) and tune endpointing + interruption thresholds on `AgentSession`.

Source: https://docs.livekit.io/agents/build/turns/

## Strategies
1. **Turn detector model** — open-weights model using VAD/STT cues for context-aware detection (recommended).
2. **Realtime model** — built-in detection from OpenAI Realtime / Gemini Live.
3. **VAD only** — start/stop from silence cues alone.
4. **STT endpointing** — phrase endpoints from STT (AssemblyAI recommended).
5. **Manual** — disable auto detection entirely.

## Setup

```python
from livekit.agents import AgentSession, TurnHandlingOptions
from livekit.plugins.turn_detector.multilingual import MultilingualModel
from livekit.plugins import silero

session = AgentSession(
    turn_handling=TurnHandlingOptions(
        turn_detection=MultilingualModel(),
    ),
    vad=silero.VAD.load(),
)
```

## Interruption knobs
- `interruption.enabled: bool`
- `interruption.mode`: `"adaptive"` (default) or `"vad"`
- `min_duration` — min speech length to count
- `min_words` — min transcribed words (STT-dependent)

## Endpointing
- Fixed: `min_delay`, `max_delay` on the turn options.
- Dynamic (Python only): adapts delay from session pause statistics.

## False interruption recovery
- `false_interruption_timeout` — silence required before emitting the event.
- `resume_false_interruption` — resume agent speech after timeout if no real input.

## Manual control

```python
session = AgentSession(turn_detection="manual", ...)

await session.interrupt()           # stop agent speech
await session.commit_user_turn()    # process buffered user input
await session.clear_user_turn()     # discard
```

## State events
Listen for `user_state_changed` / `agent_state_changed`. States: `speaking`, `listening`, `thinking`, `idle`.

## Tags
[[LiveKit]] [[Agents]] [[TurnDetection]] [[VAD]] [[Interruptions]]
