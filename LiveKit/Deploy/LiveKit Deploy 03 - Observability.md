# 03 — Observability

🔑 LiveKit Cloud unifies transcripts, traces, logs, and audio recordings per session in one timeline; data is retained 30 days and recording is per-session opt-in via the `record` parameter.

Source: https://docs.livekit.io/agents/ops/observability/ (also reachable at https://docs.livekit.io/agents/observability/)

## What's Captured

| Stream | Content |
|--------|---------|
| Transcripts | Turn-by-turn user + agent, tool calls, handoffs, metadata |
| Traces | Spans across STT → LLM → TTS pipeline, durations, token counts, speech IDs |
| Metrics | TTFT, end-of-utterance latency, interruption count, token usage |
| Logs | Stdout from the agent process at configured log level |
| Audio | Agent + user PCM, uploaded post-session (post-noise-cancellation if NC enabled) |

## Requirements

- Python SDK `livekit-agents>=1.3.0` or Node.js `@livekit/agents>=1.0.18`.
- Feature toggled on in LiveKit Cloud project settings.
- Works for agents on Cloud or self-hosted agents pointed at Cloud media servers.

## Per-Session Control

```python
from livekit.agents import AgentSession, RecordingOptions

session = AgentSession(
    ...,
    record=RecordingOptions(
        audio=True,
        transcripts=True,
        traces=True,
        logs=True,
    ),
)
```

Pass `record=False` to disable everything for sensitive calls.

## Retention

- Paid plans: hard 30-day delete.
- Free "Build" plan: enrolled in model improvement, may retain longer.

## Self-Hosted Telemetry

For fully self-hosted setups, emit OpenTelemetry from the agent and scrape Prometheus from the SFU:

```python
from livekit.agents import metrics

@session.on("metrics_collected")
def _on_metrics(ev: metrics.MetricsCollectedEvent) -> None:
    metrics.log_metrics(ev.metrics)  # also publishable to Otel
```

LiveKit server exposes Prometheus on `:6789/metrics` when `prometheus.port` is configured.

## Logging Conventions

Module-level logger, f-strings, structured fields via `extra`:

```python
import logging
logger = logging.getLogger(__name__)
logger.info(
    f"turn complete duration={duration_ms}ms",
    extra={"room": room_name, "agent": agent_name, "turn": turn_id},
)
```

## Tags
[[LiveKit]] [[Deploy]] [[Observability]] [[Agents]]
