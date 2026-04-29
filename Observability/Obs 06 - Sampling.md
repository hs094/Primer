# 06 — Sampling

🔑 Decide *which* traces to keep. Head sampling = cheap + dumb at the edge; tail sampling = expensive + smart in the collector.

Source: https://opentelemetry.io/docs/concepts/sampling/

## Head vs Tail
| | Head | Tail |
|---|---|---|
| **Where** | SDK (in app) | Collector (gateway) |
| **When** | At span start | After full trace assembled |
| **Sees** | Trace ID, parent decision | All spans, attributes, errors, latency |
| **Pros** | Zero overhead, deterministic | Keeps every error/slow trace |
| **Cons** | Misses rare errors | Stateful, RAM-heavy, complex |

## Head Sampling Recipes
```python
# 10% of traces, parent decision wins for downstream
import os
os.environ["OTEL_TRACES_SAMPLER"] = "parentbased_traceidratio"
os.environ["OTEL_TRACES_SAMPLER_ARG"] = "0.1"
```
- `always_on` / `always_off` — debug only.
- `traceidratio` — uniform fraction, no parent awareness.
- `parentbased_*` — honors upstream sampled bit (consistency across services).

## Tail Sampling (Collector)
```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - { name: errors,    type: status_code, status_code: { status_codes: [ERROR] } }
      - { name: slow,      type: latency,     latency: { threshold_ms: 1000 } }
      - { name: baseline,  type: probabilistic, probabilistic: { sampling_percentage: 5 } }
```
Result: 100% of errors + 100% of slow + 5% baseline.

## Cost vs Fidelity Cheat Sheet
- < 100 RPS: keep everything. Sampling adds complexity, saves cents.
- 100–10k RPS: head 10–50% + tail-sample errors/slow.
- 10k+ RPS: tail-sampling gateway, head 1–10%.

## Gotchas
- ⚠️ Inconsistent samplers across services = broken traces (parent in, child missing). Use `parentbased_*` everywhere.
- ⚠️ Tail sampler RAM = avg trace size × in-flight traces × `decision_wait`. Size accordingly.
- 💡 Always export 100% of **metrics** — sampling is a *trace* concern.

## Tags
[[OpenTelemetry]] [[Sampling]] [[Collector]] [[Traces]]
