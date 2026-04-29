# 02 — vLLM

🔑 vLLM is the de-facto open-source inference server: [[PagedAttention]] for KV memory + [[Continuous Batching]] for scheduling + an OpenAI-compatible HTTP server. Drop-in replacement for the OpenAI SDK with your own weights.

Source: https://docs.vllm.ai/

## Why It Wins
| Feature | What It Buys |
|---|---|
| **PagedAttention** | Block-based KV cache → near-zero fragmentation, 2–4× more concurrent requests |
| **Continuous batching** | New requests join the running batch every step, no idle GPU |
| **Chunked prefill** | Mix prefill + decode in one step → smoother TTFT under load |
| **Prefix caching** | Reuse KV for shared system prompts across requests |
| **Tensor / pipeline parallel** | Split big models across GPUs (`--tensor-parallel-size`, `--pipeline-parallel-size`) |

## Serve a Model
```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --tensor-parallel-size 2 \
  --gpu-memory-utilization 0.9 \
  --max-model-len 8192
```
Exposes `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings` on port 8000.

## Python Client (OpenAI SDK)
```python
from openai import AsyncOpenAI

client = AsyncOpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")
resp = await client.chat.completions.create(
    model="meta-llama/Llama-3.1-8B-Instruct",
    messages=[{"role": "user", "content": "Explain PagedAttention"}],
    stream=True,
)
async for chunk in resp:
    print(chunk.choices[0].delta.content or "", end="")
```

## Knobs That Matter
- ⚠️ `--max-model-len` smaller than the model's native context = bigger batches.
- `--quantization awq|gptq|fp8` for [[Quantization]] at load time.
- `--enable-prefix-caching` for repeated system prompts.
- `--guided-decoding-backend outlines` for [[Structured Outputs]].

💡 Use vLLM in-process with `LLM(...)` for offline batch jobs; use `vllm serve` for online traffic.

## Tags
[[vLLM]] [[PagedAttention]] [[Continuous Batching]] [[Tensor Parallel]] [[OpenAI API]]
