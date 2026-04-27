# GS 04 — Next Steps

🔑 Real projects split the app into a package (`proj/celery.py` + `proj/tasks.py`), use **signatures** to make tasks composable, and chain them via the **canvas** primitives — `group`, `chain`, `chord`.

## Project layout

```
src/
    proj/__init__.py
    proj/celery.py
    proj/tasks.py
```

## `proj/celery.py`

```python
from celery import Celery

app = Celery('proj',
             broker='amqp://',
             backend='rpc://',
             include=['proj.tasks'])

app.conf.update(
    result_expires=3600,
)

if __name__ == '__main__':
    app.start()
```

`include=` tells the worker which modules to import on startup so `@app.task` decorators register.

## `proj/tasks.py`

```python
from .celery import app

@app.task
def add(x, y):
    return x + y

@app.task
def mul(x, y):
    return x * y

@app.task
def xsum(numbers):
    return sum(numbers)
```

## Running

```bash
celery -A proj worker -l INFO
```

Default concurrency = number of CPU cores. Bump it for I/O-bound work; adding more than ~2x CPU count is rarely effective.

## `celery multi` — daemonized workers

```bash
celery multi start w1 -A proj -l INFO
celery multi restart w1 -A proj -l INFO
celery multi stop w1 -A proj -l INFO
celery multi stopwait w1 -A proj -l INFO   # waits for tasks to finish
```

With pid/log dirs:

```bash
mkdir -p /var/run/celery
mkdir -p /var/log/celery
celery multi start w1 -A proj -l INFO \
    --pidfile=/var/run/celery/%n.pid \
    --logfile=/var/log/celery/%n%I.log
```

Multiple workers with per-worker queue + log overrides:

```bash
celery multi start 10 -A proj -l INFO \
    -Q:1-3 images,video -Q:4,5 data \
    -Q default -L:4,5 debug
```

## Calling tasks — three forms

```python
add(2, 2)              # plain call, runs locally, returns 4
add.delay(2, 2)        # shortcut: enqueue with args
add.apply_async((2, 2))
add.apply_async((2, 2), queue='lopri', countdown=10)
```

`apply_async` is the full API — accepts `queue`, `countdown`, `eta`, `expires`, `retry`, link/link_error callbacks, etc.

## AsyncResult API

```python
res = add.delay(2, 2)
res.get(timeout=1)   # 4
res.id               # 'd6b3aea2-fb9b-4ebc-8da4-848818db9114'
res.failed()
res.successful()
res.state            # 'PENDING' | 'STARTED' | 'SUCCESS' | 'FAILURE' | 'RETRY'
```

## Signatures

A signature wraps `(task, args, kwargs, options)` so it can be passed around and chained.

```python
add.signature((2, 2), countdown=10)
add.s(2, 2)                  # short form
```

**Partial** — bind some args now, supply the rest later:

```python
s2 = add.s(2)
res = s2.delay(8)            # resolves to add(8, 2)
```

Override options at call time:

```python
s3 = add.s(2, 2, debug=True)
s3.delay(debug=False)
```

## Canvas primitives

⚠️ All canvas examples need a result backend configured.

**`group`** — run in parallel, get list of results:

```python
from celery import group
group(add.s(i, i) for i in range(10))().get()
# [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

g = group(add.s(i) for i in range(10))
g(10).get()                  # supplies the missing arg to each
```

**`chain`** — pipe output of one into the next:

```python
from celery import chain
chain(add.s(4, 4) | mul.s(8))().get()    # 64

g = chain(add.s(4) | mul.s(8))
g(4).get()                                # 64

(add.s(4, 4) | mul.s(8))().get()          # 64, pipe operator shorthand
```

**`chord`** — group + callback fed the group's results:

```python
from celery import chord
chord((add.s(i, i) for i in range(10)), xsum.s())().get()   # 90

(group(add.s(i, i) for i in range(10)) | xsum.s())().get()  # equivalent
```

## Routing

```python
app.conf.update(
    task_routes = {
        'proj.tasks.add': {'queue': 'hipri'},
    },
)
```

Or per-call:

```python
add.apply_async((2, 2), queue='hipri')
```

Worker subscribes to specific queues:

```bash
celery -A proj worker -Q hipri
celery -A proj worker -Q hipri,celery
```

## Timezone

```python
app.conf.timezone = 'Europe/London'
```

## Remote control & monitoring

Available on RabbitMQ, Redis, and Qpid brokers.

```bash
celery -A proj inspect active
celery -A proj inspect active --destination=celery@example.com
celery -A proj inspect --help
celery -A proj control --help
celery -A proj control enable_events
celery -A proj events --dump
celery -A proj events                    # curses monitor
celery -A proj control disable_events
celery -A proj status
```

`celery events` is the built-in dashboard; **Flower** is the popular web UI on top of the same event stream.

## Per-task knobs worth knowing

- `@app.task(ignore_result=True)` — skip backend write for fire-and-forget tasks.
- `@app.task(track_started=True)` — record STARTED state when the worker picks it up.

💡 Promote `delay()` calls to `signature()` the moment you need composition — once you're using `group`/`chain`/`chord` the rest of the canvas falls into place. Pair with [[Celery GS 03 - First Steps with Celery]] for the bare-minimum app and [[Celery GS 02 - Backends and Brokers]] for transport choice.
