# GS 03 — First Steps with Celery

🔑 A Celery app is just `Celery(name, broker=..., backend=...)` plus `@app.task` functions; the worker is a separate long-running process you launch with `celery -A <module> worker`.

## Prerequisites

Celery needs a **message broker** to ship tasks. RabbitMQ is the default; Redis works too. See [[Celery GS 02 - Backends and Brokers]].

```bash
$ sudo apt-get install rabbitmq-server
$ docker run -d -p 5672:5672 rabbitmq
$ docker run -d -p 6379:6379 redis
$ pip install celery
```

## The minimal app — `tasks.py`

```python
from celery import Celery

app = Celery('tasks', broker='pyamqp://guest@localhost//')

@app.task
def add(x, y):
    return x + y
```

The first arg to `Celery(...)` is the **main module name**. It must match the module you point `-A` at so tasks get auto-named correctly (`tasks.add`).

## Run the worker

```bash
$ celery -A tasks worker --loglevel=INFO
$ celery worker --help
$ celery --help
```

In production run it as a daemon (systemd, supervisor) — not in a foreground shell.

## Calling a task

```python
>>> from tasks import add
>>> add.delay(4, 4)
```

`.delay()` returns an `AsyncResult` and queues the task. The worker picks it up and runs it. Calling does **not** block.

## Keeping results

Results are **disabled by default**. To get return values back you must configure a result backend:

```python
app = Celery('tasks', backend='rpc://', broker='pyamqp://')
```

Or with Redis as backend, RabbitMQ as broker:

```python
app = Celery('tasks', backend='redis://localhost', broker='pyamqp://')
```

Then:

```python
>>> result = add.delay(4, 4)
>>> result.ready()
False
>>> result.get(timeout=1)
8
>>> result.get(propagate=False)   # don't re-raise on failure
>>> result.traceback
```

⚠️ `result.get()` turns the async call **synchronous** — usually a sign you should rethink the flow. Also: backends hold resources per result; you must eventually call `.get()` or `.forget()` on **every** `AsyncResult`.

## Configuration — three styles

**Direct attribute:**

```python
app.conf.task_serializer = 'json'
```

**Bulk update:**

```python
app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='Europe/Oslo',
    enable_utc=True,
)
```

**Dedicated module (recommended for non-trivial apps)** — `celeryconfig.py`:

```python
broker_url = 'pyamqp://'
result_backend = 'rpc://'

task_serializer = 'json'
result_serializer = 'json'
accept_content = ['json']
timezone = 'Europe/Oslo'
enable_utc = True
```

Loaded by:

```python
app.config_from_object('celeryconfig')
```

Verify it imports cleanly:

```bash
$ python -m celeryconfig
```

## Routing and rate limits via config

```python
task_routes = {
    'tasks.add': 'low-priority',
}

task_annotations = {
    'tasks.add': {'rate_limit': '10/m'}
}
```

Or change rate limit at runtime without restarting workers:

```bash
$ celery -A tasks control rate_limit tasks.add 10/m
```

## Troubleshooting

- **Worker won't start with permission error** on Debian/Ubuntu — symlink shared mem:
  ```bash
  # ln -s /run/shm /dev/shm
  ```
- **Tasks always PENDING** — no result backend configured, or task has `ignore_result=True`, or you're inspecting via a different `app` than the worker.
- **State never updates** — Celery doesn't mark a task SENT; default state is PENDING until the worker picks it up. Use `track_started=True` on the task to record STARTED.
- **Stale workers** — make sure the previous worker is fully shut down before starting a new one.

💡 Start with `pyamqp://` + `rpc://` for local hacking; move to a `celeryconfig.py` and a real backend ([[Celery GS 02 - Backends and Brokers]]) before the project gets big. Next: [[Celery GS 04 - Next Steps]].
