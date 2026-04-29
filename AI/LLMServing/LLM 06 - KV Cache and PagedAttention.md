# 06 — KV Cache and PagedAttention

🔑 At serving time, **KV cache memory dominates VRAM**, not weights. PagedAttention is vLLM's key insight: treat KV cache like virtual memory with fixed-size blocks instead of contiguous per-request buffers.

## Why KV Cache Hurts
For Llama-3-70B at FP16, per-token KV ≈ 2.5 MB. Per request:
- 4K context = 10 GB
- 32K context = 80 GB

Weights are fixed (~140 GB for 70B FP16). Cache scales with **concurrency × seq_len**. At 50 concurrent 4K-context users you need ~500 GB just for cache.

## The Old Way: Contiguous Allocation
Naive servers reserve `max_seq_len` of contiguous KV per request up front.
- ⚠️ **Internal fragmentation** — request asks for 4K, generates 200 tokens, 95% of the buffer wasted.
- ⚠️ **External fragmentation** — VRAM holes between buffers, can't fit a new request even with free bytes.
- ⚠️ **No sharing** — two requests with identical system prompts each pay full KV cost.

Result: GPU memory utilization for KV often <40%.

## PagedAttention
Inspired by OS virtual memory:
1. Split KV cache into fixed **blocks** (e.g. 16 tokens each).
2. Each request keeps a **block table** mapping logical position → physical block.
3. Allocate blocks on demand as the sequence grows.

| Win | How |
|---|---|
| ~Zero fragmentation | Block size << request, packed densely |
| **Prefix sharing** | Two requests with same system prompt point to the **same physical blocks** for the prefix |
| **Copy-on-write** | Beam search / parallel sampling fork blocks lazily |
| **Swapping** | Evict cold blocks to CPU RAM under pressure |

🧪 In the [[vLLM]] paper, PagedAttention raised throughput 2–4× vs HuggingFace TGI at the time, mostly from fitting more concurrent requests.

## What You Actually Tune
- `--block-size` (default 16) — smaller = less waste, more bookkeeping.
- `--gpu-memory-utilization` — fraction of VRAM vLLM may grab for cache.
- `--enable-prefix-caching` — turn on cross-request prefix sharing.

💡 If you understand why this works, [[Continuous Batching]] is the obvious next step — paged blocks make request join/leave cheap.

## Tags
[[KV Cache]] [[PagedAttention]] [[vLLM]] [[Memory Management]]
