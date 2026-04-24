# 22 — Background Tasks

🔑 Run work **after** the response is returned, in the same process. Good for short, fire-and-forget tasks.

## Basic

```python
from fastapi import BackgroundTasks

def write_notification(email: str, message: str = ""):
    with open("log.txt", "a") as f:
        f.write(f"{email}: {message}\n")

@app.post("/send-notification/{email}")
async def send(email: str, tasks: BackgroundTasks):
    tasks.add_task(write_notification, email, message="welcome")
    return {"status": "scheduled"}
```

- `add_task(func, *args, **kwargs)` — `func` can be sync or async.
- Runs after response is sent; errors there don't affect the HTTP status.

## With dependencies

```python
def write_log(msg: str): ...

async def get_q(q: str | None = None, tasks: BackgroundTasks = None):
    if q: tasks.add_task(write_log, f"query: {q}")
    return q

@app.post("/")
async def f(q: Annotated[str | None, Depends(get_q)]): ...
```

Dependencies can accept `BackgroundTasks` and schedule tasks.

## When to use something heavier

- Need retries, cross-worker durability, or a schedule → **Celery / RQ / Arq / Dramatiq**.
- Need pub-sub across services → message queue (NATS, Kafka, Redis Streams).
- Long-running inside an async loop is fine; CPU-bound → offload to a pool.

## Gotchas

- ⚠️ Background tasks die with the worker. A restart between response and task execution loses work.
- ⚠️ Sync task runs in threadpool → careful with shared state.
- ⚠️ DB sessions from the request dep may already be closed — open a fresh session inside the task.
