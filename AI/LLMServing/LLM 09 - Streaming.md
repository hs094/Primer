# 09 — Streaming

🔑 Streaming pushes tokens to the client as they're generated via **Server-Sent Events** (SSE). Cuts perceived latency dramatically — user sees first token in ~100ms instead of waiting 5s for the full response.

## SSE Wire Format
OpenAI-compatible servers ([[vLLM]], [[Ollama]], [[LiteLLM]] proxy) emit:
```
data: {"id":"...","choices":[{"delta":{"content":"Hello"},"index":0}]}

data: {"id":"...","choices":[{"delta":{"content":" world"},"index":0}]}

data: {"id":"...","choices":[{"delta":{},"finish_reason":"stop","index":0}]}

data: [DONE]
```
- `\n\n` separates events
- Final sentinel: literal `data: [DONE]`
- Each chunk has a `delta` (incremental) not a `message` (final)

## Python (httpx + manual SSE)
```python
import httpx, json

async with httpx.AsyncClient(timeout=None) as c:
    async with c.stream("POST", "http://localhost:8000/v1/chat/completions",
        json={"model": "...", "messages": [...], "stream": True}) as r:
        async for line in r.aiter_lines():
            if not line.startswith("data: "):
                continue
            payload = line[6:]
            if payload == "[DONE]":
                break
            chunk = json.loads(payload)
            print(chunk["choices"][0]["delta"].get("content", ""), end="")
```

## Python (OpenAI SDK — preferred)
```python
from openai import AsyncOpenAI
client = AsyncOpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

async for chunk in await client.chat.completions.create(
    model="...", messages=[...], stream=True,
):
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

## Partial JSON Problem
With [[Structured Outputs]] + streaming, mid-stream content is invalid JSON: `{"name": "Al`. Options:
- Buffer until the chunk parses (loses streaming benefit for JSON).
- Use a **tolerant streaming parser** (`partial-json-parser`, `json-stream`) that yields growing partial objects.
- Stream a **field at a time** by structuring the prompt to emit one key per line.

## Gotchas
- ⚠️ Tool/function calls stream as **incremental argument fragments** — concat all `delta.tool_calls[i].function.arguments` then parse once at the end.
- ⚠️ `usage` (token counts) only appears in the final chunk if `stream_options={"include_usage": True}` is set.
- 💡 Behind nginx/CDN: disable buffering (`proxy_buffering off`, `X-Accel-Buffering: no`) or chunks queue.

## Tags
[[Streaming]] [[SSE]] [[OpenAI API]] [[Partial JSON]]
