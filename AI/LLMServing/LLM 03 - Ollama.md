# 03 — Ollama

🔑 Ollama is a local-first model runner: pulls quantized [[GGUF]] weights, runs them on CPU/GPU via `llama.cpp`, exposes a REST API on `localhost:11434`. Optimized for laptops, not throughput.

Source: https://ollama.com/docs

## Mental Model
| Concept | Analog |
|---|---|
| **Model tag** (`llama3.1:8b-instruct-q4_K_M`) | Docker image |
| **Modelfile** | Dockerfile (FROM, PARAMETER, SYSTEM, TEMPLATE) |
| **Registry** | Docker Hub — `ollama.com/library` |
| **`ollama serve`** | Daemon listening on `:11434` |

## Core CLI
```bash
ollama pull llama3.1:8b           # download weights
ollama run llama3.1:8b            # interactive chat
ollama list                       # show local models
ollama serve                      # start API daemon
ollama create mybot -f Modelfile  # build custom model
```

## Modelfile Example
```
FROM llama3.1:8b
PARAMETER temperature 0.3
PARAMETER num_ctx 8192
SYSTEM "You are a concise senior engineer."
```

## Python Client
```python
from ollama import AsyncClient

client = AsyncClient(host="http://localhost:11434")
resp = await client.chat(
    model="llama3.1:8b",
    messages=[{"role": "user", "content": "Why GGUF?"}],
    stream=True,
)
async for chunk in resp:
    print(chunk["message"]["content"], end="")
```

## Hardware Acceleration
- **macOS**: Metal automatically (Apple Silicon).
- **Linux/Windows**: CUDA if NVIDIA driver present; ROCm for AMD.
- ⚠️ Falls back to CPU silently if GPU detection fails — check `ollama ps` for the `PROCESSOR` column.

💡 Ollama is great for dev loops and single-user apps. For multi-tenant production, use [[vLLM]] — Ollama lacks [[Continuous Batching]] and serves requests largely sequentially.

## Tags
[[Ollama]] [[GGUF]] [[Modelfile]] [[llama.cpp]] [[Local LLM]]
