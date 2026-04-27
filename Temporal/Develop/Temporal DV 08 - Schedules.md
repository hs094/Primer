# DV 08 — Schedules

🔑 Schedules drive recurring or future Workflow Executions on a fixed cadence. The newer Schedules API replaces the legacy Cron support and gives you full lifecycle control: create, pause, trigger, backfill, update, describe, list, and delete — all via the [[Temporal CL 12 - workflow]] [[Temporal EN 03 - Workflows]] Client.

## Imports

```python
import asyncio
from datetime import datetime, timedelta

from temporalio.client import (
    Client,
    Schedule,
    ScheduleActionStartWorkflow,
    ScheduleBackfill,
    ScheduleIntervalSpec,
    ScheduleOverlapPolicy,
    ScheduleSpec,
    ScheduleState,
    ScheduleUpdate,
    ScheduleUpdateInput,
)
```

## Create a Schedule

`Client.create_schedule()` registers a Schedule with a unique `schedule_id`. The action is a `ScheduleActionStartWorkflow` referencing your workflow's `.run` method, and the spec describes when to fire.

```python
async def main():
    client = await Client.connect("localhost:7233")
    await client.create_schedule(
        "workflow-schedule-id",
        Schedule(
            action=ScheduleActionStartWorkflow(
                YourSchedulesWorkflow.run,
                "my schedule arg",
                id="schedules-workflow-id",
                task_queue="schedules-task-queue",
            ),
            spec=ScheduleSpec(
                intervals=[ScheduleIntervalSpec(every=timedelta(minutes=2))]
            ),
            state=ScheduleState(note="Here's a note on my Schedule."),
        ),
    )
```

- `ScheduleActionStartWorkflow` — what to run.
- `ScheduleSpec` — when to run (intervals, calendars, cron strings).
- `ScheduleIntervalSpec(every=...)` — fixed cadence.
- `ScheduleState(note=...)` — optional metadata.

## Get a Handle

All lifecycle operations go through a `ScheduleHandle`.

```python
handle = client.get_schedule_handle("workflow-schedule-id")
```

## Backfill

`backfill()` retroactively runs actions for a past time window — useful when you want missed executions to run.

```python
now = datetime.utcnow()
await handle.backfill(
    ScheduleBackfill(
        start_at=now - timedelta(minutes=10),
        end_at=now - timedelta(minutes=9),
        overlap=ScheduleOverlapPolicy.ALLOW_ALL,
    ),
)
```

## Pause / Trigger

```python
await handle.pause(note="Pausing the schedule for now")
await handle.trigger()  # fire one execution outside the schedule
```

## Update

Updates use a callback that receives a `ScheduleUpdateInput` and returns a `ScheduleUpdate`. This is intentional — the server hands you the current state so the edit is conflict-free.

```python
async def update_schedule_simple(input: ScheduleUpdateInput) -> ScheduleUpdate:
    schedule_action = input.description.schedule.action
    if isinstance(schedule_action, ScheduleActionStartWorkflow):
        schedule_action.args = ["my new schedule arg"]
    return ScheduleUpdate(schedule=input.description.schedule)

await handle.update(update_schedule_simple)
```

## Describe / List

```python
desc = await handle.describe()
print(f"Returns the note: {desc.schedule.state.note}")

async for schedule in await client.list_schedules():
    print(f"List Schedule Info: {schedule.info}.")
```

`describe()` returns a `ScheduleDescription`.

## Delete

```python
await handle.delete()
```

⚠️ Deleting a Schedule does not affect Workflows that were already started by it — they keep running.

## Alternatives

### Cron Schedule (legacy)

```python
result = await client.execute_workflow(
    CronWorkflow.run,
    id="your-workflow-id",
    task_queue="your-task-queue",
    cron_schedule="* * * * *",
)
```

⚠️ Cron support is **not recommended** — use the Schedules API instead.

### Start Delay (one-shot)

For a single deferred execution, skip Schedules entirely:

```python
result = await client.execute_workflow(
    YourWorkflow.run,
    "your name",
    id="your-workflow-id",
    task_queue="your-task-queue",
    start_delay=timedelta(hours=1, minutes=20, seconds=30),
)
```

See also: [[Temporal CL 08 - schedule]], [[Temporal DV 09 - Timers]], [[Temporal DV 02 - Workflow Basics]].

💡 **Takeaway:** Schedules are first-class objects with a Handle-based lifecycle. Reach for `client.create_schedule(...)` + `ScheduleSpec` for recurring runs, `start_delay` for a single deferred run, and avoid `cron_schedule` in new code.
