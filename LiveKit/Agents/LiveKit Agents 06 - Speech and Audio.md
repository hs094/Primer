# 06 — Speech and Audio

🔑 `session.say()` and `session.generate_reply()` both return a `SpeechHandle` you can await or cancel; instant connect and preemptive generation cut perceived latency.

Source: https://docs.livekit.io/agents/build/audio/

## Latency features
- **Instant Connect** — captures mic input before the agent finishes joining; pre-connect audio is forwarded as `lk.agent.pre-connect-audio-buffer` so the first turn has full context
- **Preemptive speech generation** — LLM begins drafting once STT emits a final transcript, without waiting for VAD end-of-turn. Default-on for pipeline agents.

## `session.say()` — predefined utterances
```python
handle = await session.say(
    "Hold on while I look that up.",
    allow_interruptions=True,
    add_to_chat_ctx=True,   # include in LLM history
)
await handle.wait_for_playout()
```
Also accepts pre-synthesized audio (skip TTS) or text-only.

## `session.generate_reply()` — dynamic LLM turn
```python
handle = session.generate_reply(
    instructions="Summarize the last user message in one sentence.",
)
await handle.wait_for_playout()
if handle.interrupted:
    ...
```
The `instructions` argument is *appended* to the agent's session-level instructions, not a replacement.

User text can be injected manually:
```python
session.generate_reply(user_input="Book the 9 a.m. slot.")
```

## `SpeechHandle`
- `await handle.wait_for_playout()` — block until TTS audio finishes
- `handle.interrupted: bool` — was the user's barge-in honored?
- `session.current_speech` — the in-flight handle, useful to chain follow-ups after the agent stops talking

## Pattern: speak, then act
```python
handle = await session.say("Booking now...")
await handle.wait_for_playout()
await book_appointment(...)
```

## Tags
[[LiveKit]] [[Agents]] [[Audio]] [[STT]] [[TTS]] [[VAD]]
