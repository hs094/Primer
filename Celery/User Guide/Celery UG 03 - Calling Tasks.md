# UG 03 — Calling Tasks

🔑 `delay()` is the shortcut; `apply_async()` is the full API — only it accepts execution options like `countdown`, `eta`, `expires`, `link`, `queue`.

## Three ways to invoke

```python
add.delay(2, 2)                                   # shortcut, queues via worker
add.apply_async(args=[2, 2], kwargs={'z': 3})     # full API
add(2, 2)                                          # direct call, runs in-process, no message
```

`delay(*args, **kwargs)` ≈ `apply_async(args, kwargs)` — but it cannot pass execution options. Direct calling skips the broker entirely.

Both `delay` and `apply_async` return an `AsyncResult` (assuming a result backend is configured — see [[Celery UG 02 - Tasks]]).

## Scheduling

### Countdown

```python
add.apply_async((2, 2), countdown=3)   # run in 3 seconds
```

### ETA

```python
from datetime import datetime, timedelta, timezone

tomorrow = datetime.now(timezone.utc) + timedelta(days=1)
add.apply_async((2, 2), eta=tomorrow)
```

### Expiration

```python
add.apply_async((10, 10), expires=60)              # seconds
add.apply_async((10, 10), expires=tomorrow)        # datetime
```

If the worker hasn't picked it up by then, the task is revoked.

## Callbacks (`link`) and errbacks (`link_error`)

Parent return value becomes the first arg to the linked signature:

```python
add.apply_async((2, 2), link=add.s(16))            # → add(4, 16)
```

Errback receives `(request, exc, traceback)`:

```python
@app.task
def error_handler(request, exc, traceback):
    print(f'Task {request.id} raised exception: {exc!r}')

add.apply_async((2, 2), link_error=error_handler.s())
```

A list runs callbacks in order:

```python
add.apply_async((2, 2), link=[add.s(16), other_task.s()])
```

See [[Celery UG 04 - Canvas - Designing Workflows]] for `.s()` signatures.

## Message-send retries

Distinct from task-level retry: this controls re-publishing if the broker connection drops.

```python
add.apply_async((2, 2), retry=False)
```

```python
add.apply_async((2, 2), retry=True, retry_policy={
    'max_retries': 3,
    'interval_start': 0,
    'interval_step': 0.2,
    'interval_max': 0.2,
    'retry_errors': None,
})
```

Policy keys: `max_retries`, `interval_start`, `interval_step`, `interval_max`, `retry_errors`.

## Serialization & compression

```python
add.apply_async((10, 10), serializer='json')       # json (default), pickle, yaml, msgpack
add.apply_async((2, 2),   compression='zlib')      # brotli, bzip2, gzip, lzma, zlib, zstd
```

## Routing

Pick a queue:

```python
add.apply_async((2, 2), queue='priority.high')
```

Or full AMQP routing:

```python
add.apply_async(
    (2, 2),
    exchange='my_exchange',
    routing_key='routing.key',
    priority=255,        # 0–255, 255 highest
)
```

## Per-call result handling

```python
result = add.apply_async((1, 2), ignore_result=True)
```

Stream raw broker messages while waiting:

```python
def on_raw_message(body):
    print(body)

r = task.apply_async(args=(a, b))
r.get(on_message=on_raw_message, propagate=False)
```

Set `result_extended = True` in config to get task name, args, kwargs back from the result backend.

## Reusing a connection

For tight loops, hold the publisher to avoid reconnect churn:

```python
with add.app.pool.acquire(block=True) as connection:
    with add.get_publisher(connection) as publisher:
        for i, j in pairs:
            add.apply_async((i, j), publisher=publisher)
```

## `apply_async` options cheat-sheet

| Option | Purpose |
| --- | --- |
| `args`, `kwargs` | task arguments |
| `countdown`, `eta` | delayed execution |
| `expires` | revoke if not started by then |
| `retry`, `retry_policy` | broker-publish retry |
| `link`, `link_error` | callbacks / errbacks |
| `queue`, `exchange`, `routing_key`, `priority` | routing |
| `serializer`, `compression` | wire format |
| `headers` | custom AMQP headers |
| `ignore_result` | skip result backend write |

💡 Anything beyond fire-and-forget needs `apply_async`. Compose the result with signatures in [[Celery UG 04 - Canvas - Designing Workflows]].
