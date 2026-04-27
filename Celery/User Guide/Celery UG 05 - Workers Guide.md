# UG 05 — Workers Guide

🔑 A worker is a long-running process that pulls tasks off the broker and runs them in a pool. Control it via signals or the broadcast control bus (`celery inspect` / `celery control`).

## Starting / multiple workers

```bash
celery -A proj worker -l INFO
celery -A proj worker -l INFO --concurrency=10 -n worker1@%h
celery -A proj worker -l INFO --concurrency=10 -n worker2@%h
```

Hostname format: `%h` full hostname, `%n` name only, `%d` domain only.

## Stopping & restarting

`TERM` = warm shutdown (finish in-flight tasks). `KILL` may lose tasks unless `acks_late` is set. Force-terminate:

```bash
pkill -9 -f 'celery worker'
```

Restart via `celery multi`:

```bash
celery multi start 1 -A proj -l INFO -c4 --pidfile=/var/run/celery/%n.pid
celery multi restart 1 --pidfile=/var/run/celery/%n.pid
```

Or signal a daemon: `kill -HUP $pid` (background daemons only).

## Signals

| Signal | Behavior |
| --- | --- |
| `TERM` | Warm shutdown — drain running tasks |
| `QUIT` | Cold shutdown — terminate immediately |
| `INT`  | During shutdown, advance to next stage |
| `USR1` | Dump traceback for all active threads |
| `USR2` | Remote debug (`celery.contrib.rdb`) |
| `HUP`  | Restart worker (daemonized only) |

Soft shutdown (between warm and cold) honors `worker_soft_shutdown_timeout`.

## Concurrency & pool

```bash
celery -A proj worker --concurrency=10 --pool=prefork
```

Pools: `prefork` (default, CPU-bound), `eventlet`, `gevent` (I/O-bound), `solo` (single-thread, debug), `threads`. Default concurrency = CPU count. Time limits work only on `prefork` and `gevent`.

## Persistent revokes

```bash
celery -A proj worker -l INFO --statedb=/var/run/celery/worker.state
celery multi start 2 -l INFO --statedb=/var/run/celery/%n.state
```

Without `--statedb`, revoked IDs are lost on restart.

## Time limits

```python
from celery.exceptions import SoftTimeLimitExceeded

@app.task
def mytask():
    try:
        do_work()
    except SoftTimeLimitExceeded:
        clean_up_in_a_hurry()
```

Adjust at runtime:

```python
app.control.time_limit('tasks.crawl_the_web', soft=60, hard=120, reply=True)
```

## Rate limits

```python
app.control.rate_limit('myapp.mytask', '200/m')
app.control.rate_limit('myapp.mytask', '200/m',
                       destination=['celery@worker1.example.com'])
```

```bash
celery -A proj control rate_limit myapp.mytask 200/m
```

## Memory / recycling (prefork)

```bash
celery -A proj worker --max-tasks-per-child=1000
celery -A proj worker --max-memory-per-child=200000
```

## Autoscaling

```bash
celery -A proj worker --autoscale=10,3
```

`max,min` — grows up to 10, never below 3.

## Queues

```bash
celery -A proj worker -l INFO -Q foo,bar,baz
```

Add / cancel consumers at runtime:

```python
app.control.add_consumer('foo', reply=True)
app.control.add_consumer('baz', exchange='ex', exchange_type='topic',
                         routing_key='media.*', reply=True,
                         destination=['w1@example.com'])
app.control.cancel_consumer('foo', reply=True)
app.control.inspect().active_queues()
```

See [[Celery UG 08 - Routing Tasks]].

## Inspect (read-only)

```python
i = app.control.inspect()
i.registered()
i.active()
i.scheduled()
i.reserved()
app.control.ping(timeout=0.5)
```

```bash
celery -A proj inspect stats
```

## Control (mutating)

```python
app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed')
app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed',
                   terminate=True, signal='SIGKILL')
app.control.revoke([
    '7993b0aa-1f0b-4780-9af0-c47c0858b3f2',
    'f565793e-b041-4b2b-9ca4-dca22762a55d',
])
app.control.revoke_by_stamped_headers({'header': 'value'})
app.control.broadcast('shutdown')
app.control.enable_events()
```

```bash
celery -A proj control revoke d9078da5-9915-40a0-bfa1-392c7bde42ed
```

## Custom control commands

```python
from celery.worker.control import control_command, inspect_command

@control_command(args=[('n', int)], signature='[N=1]')
def increase_prefetch_count(state, n=1):
    state.consumer.qos.increment_eventually(n)
    return {'ok': 'prefetch count incremented'}

@inspect_command()
def current_prefetch_count(state):
    return {'prefetch_count': state.consumer.qos.value}
```

```bash
celery -A proj control increase_prefetch_count 3
celery -A proj inspect current_prefetch_count
```

## Watch out

- ⚠️ `KILL` without `acks_late` drops in-flight messages.
- ⚠️ Time/memory limits require `prefork` (memory) or `prefork`/`gevent` (time).
- ⚠️ Revokes are in-memory unless `--statedb` is set.

💡 Use `celery inspect`/`control` as your live debugger — it's the broadcast bus, not SSH into each box.
