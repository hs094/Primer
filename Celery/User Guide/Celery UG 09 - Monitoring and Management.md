# UG 09 — Monitoring and Management

🔑 Celery exposes two channels for observation: synchronous control commands (`celery inspect`/`celery control`) and an asynchronous events stream that powers Flower, the curses monitor, and custom cameras.

## Workers / cluster status

```bash
celery -A proj status
celery -A proj inspect active
celery -A proj inspect scheduled
celery -A proj inspect reserved
celery -A proj inspect revoked
celery -A proj inspect registered
celery -A proj inspect stats
celery -A proj inspect query_task e9f6c8f0-fec9-4ae8-a8c6-cf8c8451d4f8
```

- `active` — tasks currently executing.
- `scheduled` — tasks accepted but waiting on `eta`/`countdown` (does not include periodic).
- `reserved` — prefetched tasks holding a worker slot but not yet started.
- `stats` — pool size, total tasks, broker/RTT info per worker.

All commands accept `--timeout` and `--destination=w1@host,w2@host` to target subsets:

```bash
celery -A proj inspect -d w1@e.com,w2@e.com reserved
```

## Control (mutating commands)

```bash
celery -A proj control enable_events
celery -A proj control disable_events
```

Other broadcast control verbs include `shutdown`, `pool_grow`, `pool_shrink`, `add_consumer`, `cancel_consumer`, `rate_limit`, `time_limit`, `revoke`. They are dispatched as AMQP fanout messages on a dedicated control exchange — workers respond on a reply queue, which is why every reply is per-node.

## Result + queue management

```bash
celery -A proj result -t tasks.add 4e196aa4-0141-4601-8138-7aa33db0f577
celery -A proj purge                       # all queues in CELERY_QUEUES
celery -A proj purge -Q celery,foo,bar     # specific queues
celery -A proj purge -X celery             # exclude queue
celery -A proj migrate redis://localhost amqp://localhost   # experimental
```

## Shell

```bash
celery -A proj shell                # IPython > bpython > stdlib python
celery -A proj shell --without-tasks
celery -A proj shell --ipython
```

The app is bound as `celery` and registered tasks are imported automatically.

## Events stream

Workers only emit events when explicitly enabled — start them with `--events` (or `-E`):

```bash
celery -A proj worker -l INFO --events
```

Or flip it at runtime via `celery -A proj control enable_events`. With events on, every task transition is published on the `celeryev` exchange.

### Task event types

`task-sent` (requires `task_send_sent_event`), `task-received`, `task-started`, `task-succeeded` (with runtime), `task-failed` (with exception + traceback), `task-rejected`, `task-revoked` (terminate flag, expired flag), `task-retried`.

### Worker event types

`worker-online`, `worker-heartbeat` (every minute, includes active + processed counts), `worker-offline`.

## Curses monitor

```bash
celery -A proj events            # ncurses dashboard: tasks, workers, results
celery -A proj events --dump     # raw events to stdout
celery -A proj events -c myapp.Camera --frequency=1.0   # snapshots
```

The curses screen lets you select a task and pull tracebacks/results inline.

## Programmatic event capture

```python
from celery import Celery

app = Celery(broker='amqp://')
state = app.events.State()

def announce_failed(event):
    state.event(event)
    task = state.tasks.get(event['uuid'])
    print(f"FAILED {task.name} {task.uuid} {task.info()}")

with app.connection() as connection:
    recv = app.events.Receiver(connection, handlers={
        'task-failed': announce_failed,
        '*': state.event,
    })
    recv.capture(limit=None, timeout=None, wakeup=True)
```

`State` keeps an in-memory snapshot of cluster + task graph; pair it with a periodic camera to persist.

## Flower

```bash
pip install flower
celery -A proj flower --port=5555
celery --broker=redis://guest:guest@localhost:6379/0 flower
```

Web UI on `http://localhost:5555` — task progress + history, pool resize, queue add/cancel, revoke, rate limit, OpenID auth, HTTP API for triggering tasks. Flower also exposes Prometheus metrics.

## Broker-side inspection

RabbitMQ:

```bash
rabbitmqctl list_queues name messages messages_ready messages_unacknowledged
rabbitmqctl list_queues name consumers
rabbitmqctl list_queues name memory
rabbitmqctl -q -p my_vhost list_queues name messages
```

The RabbitMQ Management Plugin (`rabbitmq-plugins enable rabbitmq_management`) gives the same data over HTTP plus a dashboard on `:15672`.

Redis:

```bash
redis-cli -h HOST -p PORT -n DB llen <queue>
redis-cli -h HOST -p PORT -n DB keys '*'      # empty lists auto-removed
redis-cli slowlog get 10                       # latency outliers
```

Munin plugins worth wiring up: `rabbitmq-munin`, `celery_tasks`, `celery_tasks_states`.

## Mailbox tuning

The control mailbox (Kombu) takes `durable` (survive broker restart) and `exclusive` (single consumer). They are mutually exclusive — never both `True`.

See [[Celery UG 06 - Workers]] for worker control internals and [[Celery UG 12 - Debugging]] for live introspection of a stuck task.

💡 Treat `inspect`/`control` as point-in-time RPC and the events stream as the audit log — different problems, different tools.
