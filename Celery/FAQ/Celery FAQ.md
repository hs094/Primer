# Celery FAQ

🔑 Most "Celery is broken" reports are one of three things: no result backend configured, the worker is running a stale code version, or `acks_late` semantics are misunderstood.

## What should I use Celery for?

Background work after a web request returns, scheduled / periodic jobs, retry-with-backoff for flaky external calls, and fan-out work across multiple machines. If a request handler does anything you wouldn't want a user waiting on, it's a Celery task.

## Is Celery for Django only?

No — Celery works with any Python framework or none at all. Django integration is just one set of conventions; see [[Celery with Django]].

## Do I have to use RabbitMQ?

No. Redis, SQS, and Qpid are all supported. RabbitMQ is recommended when you need strict delivery guarantees; Redis is fine for most workloads. See [[Celery GS 02 - Backends and Brokers]].

## Is Celery dependent on pickle?

No. The default serializer is JSON since 4.0. Celery supports JSON, YAML, msgpack, and pickle — pickle is opt-in and a security risk if your broker is shared.

## Why is my task always `PENDING`?

`PENDING` is the default state for any task ID Celery has never heard of — it does not mean "queued". Causes:

1. No `result_backend` configured.
2. The task is being called against a different `Celery` app than the worker is using (different `broker_url` or different module path).
3. The task has `ignore_result=True`.
4. The worker isn't actually running, or is consuming a different queue.

Test with `app.control.inspect().active()` to see whether the worker is reachable.

## Why won't my task run at all?

Usually a syntax error or import failure inside `tasks.py` that prevents autodiscovery. Test from a shell:

```python
from myapp.tasks import mytask
mytask.delay()
```

If the import fails the worker silently doesn't register the task. Check worker startup logs for `Registered tasks:`.

## Can I call a task by name (without importing it)?

```python
app.send_task('app.tasks.add', args=[2, 2], kwargs={})
```

This is the only option from non-Python clients (any AMQP/Redis client can publish a Celery message).

## Can I get the current task's ID inside the task?

```python
@app.task(bind=True)
def mytask(self):
    cache.set(self.request.id, 'running')
```

`bind=True` makes the task method take `self` first, exposing `self.request` (id, args, retries, hostname, queue, etc.).

## Can I specify a custom task ID?

```python
mytask.apply_async(args=[1, 2], task_id='my-custom-id')
```

Useful for idempotency keys.

## Can I cancel / revoke a task?

```python
result.revoke()                          # via the AsyncResult
app.control.revoke(task_id)              # by id
app.control.revoke(task_id, terminate=True, signal='SIGKILL')  # mid-execution
```

`terminate=True` only works against running tasks and only on prefork workers.

## How do I purge all waiting tasks?

```python
from proj.celery import app
app.control.purge()
```

Or from the CLI: `celery -A proj purge`. This is destructive — all pending messages on the default queue are dropped.

## Should I use `retry` or `acks_late`?

- `retry()` — for expected Python errors (network blips, rate limits). The task itself decides to re-enqueue with backoff.
- `acks_late=True` — so a task that crashed mid-execution (segfault, OOM, machine power loss) gets redelivered to another worker.

Combine both **only if your task is idempotent**. Otherwise `acks_late` can produce duplicate side effects.

## Why is my task running twice?

Most common: Redis/SQS broker `visibility_timeout` is shorter than the task's runtime. The broker thinks the worker died and redelivers. Raise `broker_transport_options['visibility_timeout']` above your worst-case task duration.

Other cause: `acks_late=True` plus a worker restart — that's redelivery working as designed; make the task idempotent.

## How do I run only one task at a time / serialize execution?

Run a single worker with `--concurrency=1` consuming a dedicated queue, and route the task there:

```python
task_routes = {'app.tasks.exclusive': {'queue': 'singleton'}}
```

```bash
celery -A proj worker -Q singleton --concurrency=1
```

For finer locking, use a Redis lock (`SET NX EX`) inside the task body.

## Should I use UTC?

Yes. Keep `enable_utc=True` and write all `crontab(...)` entries assuming UTC, or set `timezone = 'Your/Zone'`. Mixing local time + DST + Celery beat is a category of bug you don't want.

## How do I retry with exponential backoff?

```python
@app.task(bind=True, autoretry_for=(IOError,), retry_backoff=True,
          retry_backoff_max=600, retry_jitter=True, max_retries=5)
def fetch(self, url):
    ...
```

`retry_backoff=True` doubles delay each attempt; `retry_jitter` randomizes to avoid thundering herds.

💡 When in doubt: check the worker logs first, your `broker_url` second, and the result backend third. The framework rarely lies — but it does default to silence. See [[Celery UG 12 - Debugging]] and [[Celery UG 05 - Workers Guide]].
