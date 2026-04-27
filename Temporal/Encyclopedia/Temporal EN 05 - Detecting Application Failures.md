# EN 05 — Detecting Application Failures

🔑 **In Temporal, timeouts detect application failures and Retry Policies mitigate them — that's the whole loop.**

## The Core Idea

*In Temporal, timeouts detect application failures. The system can then automatically mitigate these failures through retries.*

Both Workflows and Activities have dedicated timeout configurations and can be configured with a Retry Policy. The Service is the timeout enforcer; Workers are the retry executors.

## Workflow-Level Timeouts

Configured when starting the Workflow Execution:

- **Workflow Execution Timeout** — outer bound across the entire Workflow Execution Chain (all Continue-As-New + retries). Default: unlimited.
- **Workflow Run Timeout** — bound on a single Workflow Run.
- **Workflow Task Timeout** — max time a single Workflow Task may run on a Worker before the Service reschedules it. Default: 10s.

Workflows do **not** retry by default — *Workflows must be deterministic and are not intended to perform failure-prone operations*. Configure a Retry Policy at start time if you want one.

## Activity-Level Timeouts

Configured when scheduling each Activity:

| Timeout | Measures | Default | Triggers retries? |
|---|---|---|---|
| Schedule-To-Start | task in queue → worker pickup | infinite | no |
| Start-To-Close | one attempt's runtime | none (set this) | yes |
| Schedule-To-Close | total schedule → final close, across retries | infinite | n/a (caps total) |
| Heartbeat | gap between heartbeats | none | yes |

Quotes from the docs:

- Schedule-To-Start *detect[s] whether an individual Worker has crashed* or *whether the fleet of Workers polling the Task Queue is not able to keep up*. *This timeout does not trigger any retries.*
- Start-To-Close: *the Temporal Server relies on the Start-To-Close Timeout to force Activity retries* when Worker communication fails. On expiry an `ActivityTaskTimedOut` event is recorded.
- Schedule-To-Close *can be used to control the overall duration of an Activity Execution in the face of failures...without altering the Maximum Attempts field of the Retry Policy*.
- Heartbeat Timeout is *the maximum time between Activity Heartbeats*.

See [[Temporal EN 04 - Activities]] for the full activity model.

## Activity Heartbeats

For Activities that run longer than a few seconds:

- Worker calls `heartbeat(details)` periodically.
- SDK throttles network calls automatically.
- `details` are **checkpointed** — the next retry attempt sees them, enabling resume-from-progress instead of restart-from-zero.

If no heartbeat arrives within Heartbeat Timeout → Activity attempt fails → retry per policy.

## Retry Policies

A **Retry Policy** is *a collection of settings that tells Temporal how and when to try again after something fails in a Workflow Execution or Activity Task Execution*.

| Field | Default | Role |
|---|---|---|
| Initial Interval | 1s | wait before first retry |
| Backoff Coefficient | 2.0 | multiplier per retry |
| Maximum Interval | 100× Initial (100s) | cap on per-retry wait |
| Maximum Attempts | unlimited (Activity) / 1 (Workflow) | hard attempt cap |
| Non-Retryable Errors | none | error types that bypass retry entirely |

Backoff sequence with defaults: 1s, 2s, 4s, 8s, 16s, 32s, 64s, 100s, 100s, 100s…

## Failure Classification

```
            Activity raises error
                      │
       ┌──────────────┴──────────────┐
       ▼                             ▼
  Non-Retryable?                  Retryable
       │                             │
       ▼                             ▼
  fail Activity            schedule retry per
  immediately               Retry Policy until
  → bubbles to              Maximum Attempts /
  Workflow                  Schedule-To-Close
```

Mark domain errors (`InvalidInput`, `NotFound`, `BusinessRuleViolation`) as **Non-Retryable** so the workflow learns about them quickly instead of retry-spinning. See [[Temporal RF 05 - Failures]].

## Cross-Refs

- [[Temporal EN 04 - Activities]] — where timeouts live
- [[Temporal EN 03 - Workflows]] — workflow timeouts
- [[Temporal EN 06 - Workers]] — task pickup behavior
- [[Temporal EN 07 - Event History]] — `*TimedOut` events
- [[Temporal RF 04 - Errors]], [[Temporal RF 05 - Failures]]

⚠️ Forgetting `Start-To-Close` is the single most common mistake. Without it, a hung activity holds the slot indefinitely — the Service has no signal to schedule a retry.

⚠️ Schedule-To-Close caps *total* time including retries. Setting it short with aggressive retries effectively reduces your attempt budget. Pick one knob (`MaximumAttempts` *or* `ScheduleToClose`) and own it.

💡 **Takeaway:** detect = timeouts, mitigate = retries. Always set Start-To-Close, mark permanent errors non-retryable, heartbeat long activities.
