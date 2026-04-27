# UG 02 — Tasks

🔑 A task is a class built from your function by `@app.task`; `bind=True` exposes `self` so you get `self.request`, `self.retry`, and lifecycle handlers.

## The decorator

```python
@app.task
def create_user(username, password):
    User.objects.create(username=username, password=password)
```

Options as decorator kwargs:

```python
@app.task(serializer='json')
def create_user(username, password):
    User.objects.create(username=username, password=password)
```

## Names

Every task has a unique name. Auto-derived from module + function unless overridden:

```python
@app.task(name='tasks.add')
def add(x, y):
    return x + y

add.name   # 'tasks.add'
```

Worker and caller must agree on the name — use [[Celery UG 01 - Application]]'s main-module convention or set `name=` explicitly.

## Bound tasks (`bind=True`)

```python
@app.task(bind=True)
def add(self, x, y):
    logger.info(self.request.id)
```

`self` is the task instance — required for retries, request context, custom state.

## Request context

`self.request` (a `Context`) exposes:

- `id` — unique task id
- `args`, `kwargs` — invocation arguments
- `retries` — current retry count (starts at 0)
- `is_eager` — running locally, not via worker
- `hostname` — worker node name
- `called_directly` — task wasn't dispatched through a worker
- `delivery_info` — exchange + routing key

## Retries

### Manual

```python
@app.task(bind=True)
def send_twitter_status(self, oauth, tweet):
    try:
        twitter = Twitter(oauth)
        twitter.update_status(tweet)
    except (Twitter.FailWhaleError, Twitter.LoginError) as exc:
        raise self.retry(exc=exc)
```

`self.retry()` raises — code after it never runs.

### Per-call retry delay

```python
@app.task(bind=True, default_retry_delay=30 * 60)
def add(self, x, y):
    try:
        something_raising()
    except Exception as exc:
        raise self.retry(exc=exc, countdown=60)
```

### Auto-retry on listed exceptions

```python
@app.task(autoretry_for=(FailWhaleError,),
          retry_kwargs={'max_retries': 5})
def refresh_timeline(user):
    return twitter.refresh_timeline(user)
```

### Exponential backoff

```python
@app.task(autoretry_for=(RequestException,), retry_backoff=True)
def x():
    ...
```

Retry knobs:

- `autoretry_for` — tuple of exception classes to retry
- `dont_autoretry_for` — exceptions to skip even if matched above
- `max_retries` — default 3, `None` = unlimited
- `retry_backoff` — bool or seed seconds for exponential backoff
- `retry_backoff_max` — cap (default 600s)
- `retry_jitter` — add randomness (default `True`)

## Lifecycle handlers

Order: `before_start` → `run` → result stored → one of `on_success` / `on_retry` / `on_failure` → `after_return` (skipped on RETRY/REJECTED/IGNORED).

```python
from celery import Task

class MyTask(Task):
    def before_start(self, task_id, args, kwargs):
        print(f"Task {task_id} starting with args {args}")

    def on_success(self, retval, task_id, args, kwargs):
        print(f"Task {task_id} succeeded: {retval}")

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        print(f"Task {task_id} failed: {exc}")

    def after_return(self, status, retval, task_id, args, kwargs, einfo):
        print(f"Task {task_id} finished: {status}")

@app.task(base=MyTask)
def my_task(x, y):
    return x + y
```

## States

`PENDING` (waiting or unknown) → `STARTED` (only with `track_started=True`) → `SUCCESS` | `FAILURE` | `RETRY` | `REVOKED`. Custom states are fine:

```python
@app.task(bind=True)
def upload_files(self, filenames):
    for i, file in enumerate(filenames):
        if not self.request.called_directly:
            self.update_state(state='PROGRESS',
                meta={'current': i, 'total': len(filenames)})
```

⚠️ `PENDING` means *unknown* — Celery doesn't record SENT. An unknown task id is also PENDING.

## Custom Task base class

```python
import celery

class MyTask(celery.Task):
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        print(f'{task_id!r} failed: {exc!r}')

@app.task(base=MyTask)
def add(x, y):
    raise KeyError()
```

## Common options

```python
@app.task(
    name='explicit.task.name',
    bind=True,
    ignore_result=False,
    max_retries=3,
    default_retry_delay=180,
    rate_limit='100/m',
    time_limit=300,
    soft_time_limit=250,
    serializer='json',
    compression='gzip',
    track_started=True,
    acks_late=True,
)
def my_task(self):
    ...
```

- `ignore_result` — skip backend write
- `rate_limit` — per-worker, e.g. `'100/m'`, `'10/s'`
- `time_limit` / `soft_time_limit` — hard kill / `SoftTimeLimitExceeded`
- `acks_late` — ack after run; requires idempotent tasks
- `track_started` — emit STARTED state

## Stacking decorators

```python
@app.task
@decorator2
@decorator1
def add(x, y):
    return x + y
```

`@app.task` must be outermost (first listed) so it wraps the final composed function.

💡 `bind=True` unlocks the interesting half of Celery; reach for it whenever you need retries, state, or context. Calling these tasks → [[Celery UG 03 - Calling Tasks]].
