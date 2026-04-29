# 07 — Continuous Batching

🔑 Continuous (a.k.a. **iteration-level**) batching schedules at every decode step, not every request. New requests join the running batch instantly; finished ones leave instantly. This is the single biggest production throughput lever after [[PagedAttention]].

## Static vs Dynamic vs Continuous
| Strategy | When batch forms | Problem |
|---|---|---|
| **Static** | Offline, fixed shape | Need all requests up front |
| **Dynamic** (request-level) | At request arrival, fixed for run | Slow request blocks fast ones until everyone finishes |
| **Continuous** (iteration-level) | Re-evaluated every step | None — this is what [[vLLM]], TGI, TensorRT-LLM do |

## Why It Wins
A typical generation has:
- Long-tail length distribution (some 50 tokens, some 2000)
- Mixed prefill (compute-heavy) + decode (memory-heavy)

With request-level batching, a 50-token request waits for the 2000-token one to finish — GPU sits at low utilization. With continuous batching, the slot freed by the short request is filled in the next step.

🧪 Throughput gain over static batching: typically **5–10×** in real workloads with mixed lengths.

## How vLLM Schedules a Step
1. Pick a set of running decode requests (still generating).
2. Optionally chunk a prefill request and **mix it into the same step** (chunked prefill).
3. Run one forward pass producing one token per running request.
4. Append tokens, free finished requests' [[KV Cache]] blocks, admit new ones.

## Implications for Clients
- ⚠️ Your p50 latency depends on **how loaded the server is**, not just your prompt — batch-mate count varies per step.
- 💡 Long prompts hurt everyone briefly (prefill chunk dominates a step). Keep system prompts short or use [[PagedAttention]] prefix caching.
- Concurrency is set by **KV cache capacity**, not CPU threads. `gpu-memory-utilization` is the real concurrency dial.

## Knobs in vLLM
- `--max-num-seqs` — hard cap on concurrent sequences.
- `--max-num-batched-tokens` — total tokens per step (prefill + decode).
- `--enable-chunked-prefill` — mix prefill into decode steps for smoother TTFT.

## Tags
[[Continuous Batching]] [[Scheduling]] [[Throughput]] [[vLLM]] [[Chunked Prefill]]
