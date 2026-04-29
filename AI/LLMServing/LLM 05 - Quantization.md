# 05 — Quantization

🔑 Quantization shrinks weights from FP16 → INT8/INT4/FP8 so a 70B model fits where only a 13B did. You trade a few % accuracy for 2–4× memory savings and often faster decode (more tokens/sec/GB of bandwidth).

## Format Cheatsheet
| Format | Bits | Where | Trade-off |
|---|---|---|---|
| **FP16 / BF16** | 16 | Training, top-quality serving | Baseline |
| **FP8** | 8 | H100/H200, [[vLLM]] `--quantization fp8` | Near-lossless, needs Hopper+ |
| **INT8** | 8 | Older GPUs, SmoothQuant | Small accuracy hit |
| **GPTQ** | 4 | Post-training, weight-only | ~1% accuracy loss, fast |
| **AWQ** | 4 | Activation-aware, weight-only | Better than GPTQ on instruct models |
| **GGUF** (Q4_K_M, Q5_K_M, …) | 2–8 | [[Ollama]] / `llama.cpp` | CPU+GPU, mixed precision per layer |

## What Gets Quantized
- **Weight-only** (GPTQ, AWQ, GGUF): activations stay FP16, only weights compressed → easy decode speedup.
- **Weight + activation** (INT8 SmoothQuant, FP8): both compressed → bigger speedup, harder to keep accuracy.
- **KV cache quantization** ([[KV Cache]] in INT8/FP8): orthogonal — bigger batches at the same VRAM.

## Choosing
- ⚠️ Test on **your eval set**, not the model card. Coding/math degrade faster than chat.
- 4-bit AWQ is the default sweet spot for serving 7B–70B on consumer/single-GPU.
- FP8 if you have H100s — keeps near-FP16 quality and unlocks tensor-core throughput.
- GGUF only if running on `llama.cpp` / [[Ollama]] / CPU.

## Loading in vLLM
```bash
vllm serve TheBloke/Llama-3.1-70B-Instruct-AWQ --quantization awq
```

🧪 Always run `lm-eval-harness` (or your task evals) on the quantized model before shipping. A 1% MMLU drop can be a 10% drop on your specific task.

💡 Quantization is the cheapest lever: same hardware, 2–4× capacity, hours not weeks.

## Tags
[[Quantization]] [[GPTQ]] [[AWQ]] [[GGUF]] [[FP8]]
