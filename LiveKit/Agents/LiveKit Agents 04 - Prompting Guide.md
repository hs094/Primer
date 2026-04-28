# 04 — Prompting Guide

🔑 Voice agents are blind to the pipeline they sit in — write prompts that produce TTS-friendly plain text and explicitly cover identity, tools, goals, and guardrails.

Source: https://docs.livekit.io/agents/start/prompting/ (canonical; `/agents/build/prompting/` 404s)

## Five sections every prompt should have
1. **Identity** — open with `You are <name>, <role>...`. Define responsibilities up front.
2. **Output formatting** — "Respond in plain text only. Never use JSON, markdown, lists, tables, code, emojis, or other complex formatting." Spell numbers, expand acronyms.
3. **Tools** — when to call which tool, what inputs to gather first, how to narrate outcomes. "If an action fails, say so once, propose a fallback, or ask how to proceed."
4. **Goals & guardrails** — top-level objective + safety boundaries. Decline harmful/out-of-scope; offer general info on medical/legal/financial without specific advice.
5. **User information** — pull personalization from job metadata.

## Example skeleton
```python
INSTRUCTIONS = f"""
You are Aria, a scheduling assistant for {company_name}.

# Output
Respond in plain spoken English. No markdown, no lists, no emojis.
Spell out numbers ("nine a.m.", not "9am").

# Tools
- book_appointment(date, time, reason): collect all three before calling.
  After booking, confirm the date and time aloud.
- cancel_appointment(id): require explicit user confirmation first.

# Goals
Help the user schedule, reschedule, or cancel appointments.

# Guardrails
Decline medical advice. Stay on scheduling topics.
If unsure, ask one clarifying question.

# User
Name: {user_name}. Timezone: {user_tz}.
"""

class Assistant(Agent):
    def __init__(self) -> None:
        super().__init__(instructions=INSTRUCTIONS)
```

## Iterating
- Test with the built-in pytest harness for Python agents
- Use LiveKit Cloud observability to surface bad turns; refine instructions from real transcripts

## Tags
[[LiveKit]] [[Agents]] [[Prompting]] [[VoiceAI]] [[LLM]]
