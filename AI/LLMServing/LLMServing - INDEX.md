# LLM Serving — INDEX

🔑 How to take open-weights models and serve them at production latency, cost, and throughput. Inference internals, the dominant servers ([[vLLM]], [[Ollama]]), and the gateway layer ([[LiteLLM]]) that sits in front of everything.

## Read Order
1. [[LLM 01 - Inference Basics]] — prefill vs decode, KV cache, latency vs throughput.
2. [[LLM 06 - KV Cache and PagedAttention]] — why KV memory dominates, how vLLM solves it.
3. [[LLM 07 - Continuous Batching]] — iteration-level scheduling, the throughput unlock.
4. [[LLM 02 - vLLM]] — the server that puts 1–3 together.
5. [[LLM 05 - Quantization]] — INT8/INT4/FP8/GGUF/AWQ/GPTQ trade-offs.
6. [[LLM 03 - Ollama]] — local-first runner for laptops and dev loops.
7. [[LLM 09 - Streaming]] — SSE, OpenAI streaming shape, partial JSON.
8. [[LLM 08 - Structured Outputs]] — JSON mode, function calling, constrained decoding.
9. [[LLM 04 - LiteLLM]] — unified API + proxy across 100+ providers.
10. [[LLM 10 - Gateway Patterns]] — when and why to put a gateway in front.

## Map of Concerns
| Layer | Notes |
|---|---|
| **Inference internals** | 01, 06, 07 |
| **Servers** | 02 (vLLM), 03 (Ollama) |
| **Memory / cost** | 05 (quantization), 06 (KV cache) |
| **Client interface** | 08 (structured), 09 (streaming) |
| **Org-level** | 04 (LiteLLM), 10 (gateway) |

## Decision Shortcuts
- 💡 **Single user, laptop**: [[Ollama]].
- 💡 **Multi-tenant production, own GPUs**: [[vLLM]] behind [[LiteLLM]] proxy.
- 💡 **Many providers, no own GPUs**: [[LiteLLM]] proxy → OpenAI/Anthropic/Bedrock/Vertex.
- 💡 **Need guaranteed JSON**: [[Structured Outputs]] via vLLM guided decoding or Outlines.
- 💡 **First token feels slow**: enable [[Streaming]]; check prefill length.
- 💡 **Out of GPU memory**: [[Quantization]] (AWQ/FP8) before buying more cards.

## Tags
[[LLM Serving]] [[vLLM]] [[Ollama]] [[LiteLLM]] [[Quantization]] [[KV Cache]] [[PagedAttention]] [[Streaming]]
