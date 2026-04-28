# 24 — LLM Integrations

🔑 The LLM is the reasoning core — it receives STT transcripts, runs tools, and emits text that the TTS speaks.

Source: https://docs.livekit.io/agents/integrations/llm/

## LiveKit Inference (managed)
Includes OpenAI GPT-5 series, Google Gemini, DeepSeek, xAI Grok, and others. No keys required; billed through LiveKit Cloud.

```python
from livekit.agents import AgentSession, inference

session = AgentSession(
    llm=inference.LLM(model="openai/gpt-5.3-chat-latest"),
)
```

## Plugins (BYO key)
15+ providers available as `livekit-agents[<name>]` extras: Anthropic, Groq, Mistral, Cerebras, Together, Fireworks, Perplexity, Ollama, vLLM, Bedrock, Vertex, etc.

```python
from livekit.plugins import anthropic, openai, google

llm = anthropic.LLM(model="claude-4-5-sonnet-latest")
llm = openai.LLM(model="gpt-4o")
llm = google.LLM(model="gemini-2.5-flash")
```

## Capabilities
- **Tool calling** — decorate methods with `@function_tool` on the `Agent`.
- **Vision** — pass image URLs or live video frames into `ChatContext`.
- **Streaming** — `LLM` returns a `ChatStream`; tokens flow as they arrive.
- **Standalone** — call `llm.chat(chat_ctx=...)` outside of an `AgentSession` for background tasks.

## Standalone Example
```python
from livekit.agents.llm import ChatContext

ctx = ChatContext().append(role="user", text="summarize today's meeting")
async with llm.chat(chat_ctx=ctx) as stream:
    async for chunk in stream:
        print(chunk.delta.content, end="")
```

## Tags
[[LiveKit]] [[Agents]] [[LLM]] [[Inference]] [[ToolCalling]] [[Plugins]]
