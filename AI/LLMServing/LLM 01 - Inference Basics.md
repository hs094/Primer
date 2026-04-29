# 01 — Inference Basics

🔑 LLM inference is two phases with completely different shapes: **prefill** (compute-bound, one big matmul over the prompt) and **decode** (memory-bound, one token at a time). Throughput tuning means knowing which phase you're in.

## Two-Phase Generation
| Phase | Work | Bottleneck | Parallelism |
|---|---|---|---|
| **Prefill** | Process all prompt tokens at once, fill [[KV Cache]] | GPU compute (FLOPs) | Batched matmul across seq_len |
| **Decode** | Generate one token, append to KV cache, repeat | Memory bandwidth (HBM reads) | Across batch only |

Decode is autoregressive — token N depends on token N-1, so seq_len parallelism is gone.

## Latency vs Throughput
- ⚠️ **TTFT** (time-to-first-token) ≈ prefill latency. Dominated by prompt length.
- **TPOT** (time-per-output-token) ≈ decode step time. Dominated by model size + memory bandwidth.
- **Throughput** = tokens/sec across all concurrent requests. Grows with batch size until memory caps you.

💡 Bigger batches help throughput but hurt per-request latency. Production servers use [[Continuous Batching]] to get both.

## KV Cache Mechanics
For each token, attention layers cache `K` and `V` projections so future tokens don't recompute them.

Memory per token ≈ `2 × num_layers × num_heads × head_dim × dtype_bytes`. For Llama-3-70B at fp16: ~2.5 MB/token. A 4K context = ~10 GB just for cache.

🧪 Quick sanity check: if your model is 14 GB but a single 4K-context request needs 10 GB of KV cache, your batch size on a 24 GB card is ~1.

## What Servers Optimize
1. KV cache memory layout → [[PagedAttention]]
2. Scheduling prefill + decode together → [[Continuous Batching]]
3. Reducing weights footprint → [[Quantization]]
4. Pushing tokens to client incrementally → [[Streaming]]

## Tags
[[LLM Serving]] [[KV Cache]] [[Prefill]] [[Decode]] [[Throughput]]
