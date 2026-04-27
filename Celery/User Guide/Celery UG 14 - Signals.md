# UG 14 — Signals

🔑 Signals are Django-style hooks fired at well-defined points (task publish, prerun, failure, worker init, beat start) — connect handlers to do cleanup, metrics, audit logging, and per-worker config without subclassing anything.

## Connection pattern

```python
from celery.signals import after_task_publish

@after_task_publish.connect
def task_sent_handler(sender=None, headers=None, body=None, **kwargs):
    info = headers if 'task' in headers else body
    print('after_task_publish for task id {info[id]}'.format(info=info,))
```

Filter by sender (task name):

```python
@after_task_publish.connect(sender='proj.tasks.add')
def task_sent_handler(sender=None, headers=None, body=None, **kwargs):
    info = headers if 'task' in headers else body
    print('after_task_publish for task id {info[id]}'.format(info=info,))
```

⚠️ Always accept `**kwargs` — Celery adds new signal arguments over time and unknown kwargs would otherwise break handlers.

## Task signals

| Signal | Sender | Key arguments |
|---|---|---|
| `before_task_publish` | task name | `body, exchange, routing_key, headers, properties, declare, retry_policy` |
| `after_task_publish` | task name | `headers, body, exchange, routing_key` |
| `task_prerun` | task object | `task_id, task, args, kwargs` |
| `task_postrun` | task object | `task_id, task, args, kwargs, retval, state` |
| `task_retry` | task object | `request, reason, einfo` (only `request` guaranteed) |
| `task_success` | task object | `result` |
| `task_failure` | task object | `task_id, exception, args, kwargs, traceback, einfo` |
| `task_internal_error` | task object | `task_id, args, kwargs, request, exception, traceback, einfo` |
| `task_received` | Consumer | `request` (Request, not `task.request`) |
| `task_revoked` | task object | `request, terminated, signum, expired` |
| `task_unknown` | Consumer | `name, id, message, exc` |
| `task_rejected` | Consumer | `message, exc` |

`task_sent` is **deprecated** — use `after_task_publish`.

## Worker signals

```python
from celery.signals import celeryd_init

@celeryd_init.connect
def configure_workers(sender=None, conf=None, **kwargs):
    if sender in ('worker1@example.com', 'worker2@example.com'):
        conf.task_default_rate_limit = '10/m'
    if sender == 'worker3@example.com':
        conf.worker_prefetch_multiplier = 0
```

Add a custom direct queue per-host:

```python
from celery.signals import celeryd_after_setup

@celeryd_after_setup.connect
def setup_direct_queue(sender, instance, **kwargs):
    queue_name = '{0}.dq'.format(sender)
    instance.app.amqp.queues.select_add(queue_name)
```

Pre-fork cleanup (e.g. close gRPC channels that don't survive `fork`):

```python
@signals.worker_before_create_process.connect
def clean_channels(**kwargs):
    grpc_singleton.clean_channel()
```

Other worker signals: `worker_init`, `worker_ready` (accepting work), `worker_shutting_down` (`sig, how, exitcode`), `worker_shutdown`, `worker_process_init` (handlers must not block more than 4 seconds), `worker_process_shutdown` (`pid, exitcode` — not guaranteed to fire), `heartbeat_sent`.

## Beat signals

- `beat_init` — fires on `celery beat` startup, sender is the Service instance.
- `beat_embedded_init` — additionally fires when beat runs embedded inside a worker (`-B`).

## Eventlet pool signals

`eventlet_pool_started`, `eventlet_pool_preshutdown`, `eventlet_pool_postshutdown`, `eventlet_pool_apply` (with `target, args, kwargs`). See [[Celery UG 13 - Concurrency]].

## Logging signals

- `setup_logging` — connecting it **fully overrides** Celery's logging setup; Celery won't configure the loggers itself.
- `after_setup_logger` — augment the global logger after Celery sets it up.
- `after_setup_task_logger` — same, per-task logger.

All three pass `loglevel, logfile, format, colorize`.

## Custom CLI options

Add a preload option, then react to it:

```python
from celery import Celery, signals
from click import Option

app = Celery()

app.user_options['preload'].add(Option(
    ('--monitoring',), is_flag=True,
    help='Enable our external monitoring utility, blahblah',
))

@signals.user_preload_options.connect
def handle_preload_options(options, **kwargs):
    if options.get('monitoring'):
        enable_monitoring()
```

Also: `import_modules` (sender = app instance) fires when Celery imports modules from configuration.

## Why use them

- **Cleanup** — close DB connections / gRPC channels around `worker_process_init` / `worker_before_create_process`.
- **Metrics** — increment counters in `task_prerun` / `task_postrun` / `task_failure`.
- **Audit** — log task publish + completion via `after_task_publish` + `task_success`.
- **Per-host config** — branch on `sender` in `celeryd_init` to set rate limits, prefetch, queues.
- **Custom dead-letter routing** — react to `task_revoked` / `task_failure` with structured logs and re-publish.

## Where to put handlers

Define them in a module that's imported on worker startup — typically `proj/celery.py` or a `signals.py` listed in `app = Celery(..., include=[...])`. Handlers in the parent process won't fire inside child processes unless they're imported there too — that's why `worker_process_init` exists.

💡 Signals are the right hook for cross-cutting concerns. Reach for them before subclassing `Task` or wrapping `apply_async`.
