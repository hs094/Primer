# Configuration and Defaults

🔑 Celery has 200+ settings; you only need ~30. They live on `app.conf` (or a module loaded via `app.config_from_object(...)`) and use lowercase `snake_case` names.

## Loading config

```python
from celery import Celery

app = Celery('proj')

# Option A — direct
app.conf.task_serializer = 'json'

# Option B — bulk
app.conf.update(task_serializer='json', accept_content=['json'])

# Option C — dedicated module (recommended)
app.config_from_object('celeryconfig')
```

For Django the conventional form is `CELERY_*` upper-case keys in `settings.py` plus `app.config_from_object('django.conf:settings', namespace='CELERY')`. See [[Celery with Django]].

## Broker

| Setting | Default | What it does |
|---|---|---|
| `broker_url` | `"amqp://"` | Broker connection URL. Accepts a list of URLs for failover. |
| `broker_connection_retry_on_startup` | `True` | Retry initial broker connection. Set `False` to fail fast. |
| `broker_connection_timeout` | `4.0` | Seconds before initial connection times out. |
| `broker_pool_limit` | `10` | Max open broker connections. `None` disables pooling. |
| `broker_heartbeat` | `120.0` | AMQP heartbeat in seconds. |
| `broker_use_ssl` | `False` | SSL config dict for the broker connection. |
| `broker_transport_options` | `{}` | Per-transport tuning, e.g. `visibility_timeout` for SQS / Redis. |
| `broker_login_method` | `"AMQPLAIN"` | AMQP auth mechanism. |

```python
broker_url = 'redis://localhost:6379/0'
broker_transport_options = {'visibility_timeout': 3600}
broker_connection_retry_on_startup = True
```

## Result backend

| Setting | Default | What it does |
|---|---|---|
| `result_backend` | `None` | Backend URL. Without it, results are discarded. |
| `result_expires` | `86400` (1 day) | Auto-delete results after N seconds. `None` disables. |
| `result_extended` | `False` | Store name, args, kwargs, worker, retries on the result. |
| `result_persistent` | `False` | (RPC/AMQP) survive broker restart. |
| `result_serializer` | `"json"` | One of `json`, `pickle`, `yaml`, `msgpack`. |
| `result_compression` | `None` | `gzip`, `bzip2`. |
| `result_accept_content` | follows `accept_content` | Whitelist of result content types. |
| `result_backend_transport_options` | `{}` | Backend-specific options. |

```python
result_backend = 'redis://localhost/1'
result_expires = 3600
result_extended = True
```

## Tasks

| Setting | Default | What it does |
|---|---|---|
| `task_serializer` | `"json"` | Default serializer for outgoing tasks. |
| `accept_content` | `{'json'}` | Whitelist of content types the worker will load. |
| `task_compression` | `None` | Per-message compression. |
| `task_routes` | `None` | Map task name → queue / exchange / routing key. |
| `task_default_queue` | `"celery"` | Queue used when no route matches. |
| `task_default_exchange` | matches queue | Default exchange. |
| `task_default_routing_key` | matches queue | Default routing key. |
| `task_default_rate_limit` | `None` | Global default rate limit (e.g. `"10/s"`). |
| `task_annotations` | `None` | Override task attrs by name (`{'tasks.add': {'rate_limit': '10/m'}}`). |
| `task_time_limit` | `None` | Hard timeout — `SIGKILL` after N seconds. |
| `task_soft_time_limit` | `None` | Raises `SoftTimeLimitExceeded` so the task can clean up. |
| `task_acks_late` | `False` | Ack after execution — required for redelivery on worker crash. |
| `task_reject_on_worker_lost` | `False` | Re-queue when a worker dies mid-task. Pair with `acks_late`. |
| `task_track_started` | `False` | Record the `STARTED` state. |
| `task_ignore_result` | `False` | Drop results to save backend churn. |
| `task_store_errors_even_if_ignored` | `False` | Keep failures even with `ignore_result`. |
| `task_send_sent_event` | `False` | Emit `task-sent` events for monitors. |
| `task_remote_tracebacks` | `False` | Include the worker traceback in `result.traceback`. Needs `tblib`. |
| `task_always_eager` | `False` | Execute tasks synchronously in the caller. Tests only. |
| `task_eager_propagates` | `False` | When eager, raise exceptions instead of returning a failed result. |

```python
task_serializer = 'json'
accept_content = ['json']

task_routes = {
    'app.tasks.heavy_*': {'queue': 'cpu'},
    'app.tasks.email_*': {'queue': 'io'},
}
task_acks_late = True
task_reject_on_worker_lost = True
task_time_limit = 300
task_soft_time_limit = 240
```

## Worker

| Setting | Default | What it does |
|---|---|---|
| `worker_concurrency` | CPU count | Number of child processes (or threads/greenlets). |
| `worker_pool` | `"prefork"` | `prefork`, `solo`, `threads`, `gevent`, `eventlet`. |
| `worker_prefetch_multiplier` | `4` | Messages fetched per slot. Set `1` for long/uneven tasks. |
| `worker_max_tasks_per_child` | `None` | Recycle process after N tasks. Plugs leaks. |
| `worker_max_memory_per_child` | `None` | Recycle when RSS exceeds N KB. |
| `worker_disable_rate_limits` | `False` | Globally ignore rate limits for speed. |
| `worker_send_task_events` | `False` | Emit per-task events (needed for Flower). |
| `worker_state_db` | `None` | Path to persist revoked-tasks list across restarts. |
| `worker_timer_precision` | `1.0` | ETA scheduler tick (seconds). |

```python
worker_concurrency = 8
worker_prefetch_multiplier = 1   # one message per slot — fair scheduling
worker_max_tasks_per_child = 1000
worker_send_task_events = True
```

⚠️ `worker_prefetch_multiplier=1` is the standard fix for long-running tasks starving fast ones. Default of 4 is tuned for short tasks.

## Beat (periodic tasks)

| Setting | Default | What it does |
|---|---|---|
| `beat_schedule` | `{}` | Dict of scheduled entries. |
| `beat_scheduler` | `"celery.beat:PersistentScheduler"` | Override for `django-celery-beat` etc. |
| `beat_schedule_filename` | `"celerybeat-schedule"` | Shelve file for the default scheduler. |
| `beat_max_loop_interval` | `0` (auto) | Max seconds between schedule polls. |
| `beat_sync_every` | `0` | Force sync to disk every N task sends. |

```python
from celery.schedules import crontab

beat_schedule = {
    'cleanup-every-night': {
        'task': 'app.tasks.cleanup',
        'schedule': crontab(hour=3, minute=0),
    },
}
```

## Security (message signing)

| Setting | Default | What it does |
|---|---|---|
| `security_key` | `None` | Path to PEM private key. |
| `security_certificate` | `None` | Path to PEM cert. |
| `security_cert_store` | `None` | Glob/dir of trusted public certs. |
| `security_digest` | `"sha256"` | Signing hash. |

Enable with `app.setup_security()` and `task_serializer = 'auth'`.

## Logging

| Setting | Default | What it does |
|---|---|---|
| `worker_hijack_root_logger` | `True` | Worker takes over root logger config. Set `False` if you own logging. |
| `worker_log_format` | Celery default | `%`-style format string. |
| `worker_task_log_format` | Celery default | Per-task log line format. |
| `worker_redirect_stdouts` | `True` | Capture `print()` from tasks into the logger. |
| `worker_redirect_stdouts_level` | `"WARNING"` | Level for redirected stdout/stderr. |

⚠️ With `worker_hijack_root_logger=True` (default), your app's logging config can get clobbered. Disable it if you call `dictConfig` yourself.

## Redis-specific

| Setting | Default | What it does |
|---|---|---|
| `redis_max_connections` | unlimited | Cap on the result-backend pool. |
| `redis_socket_timeout` | `120.0` | Read/write timeout. |
| `redis_socket_connect_timeout` | `None` | Connect timeout. |
| `redis_retry_on_timeout` | `False` | Retry on socket timeout. |
| `redis_socket_keepalive` | `False` | TCP keepalive. |
| `redis_backend_use_ssl` | `None` | SSL dict for the result backend. |

For the broker, set the same options under `broker_transport_options` (e.g. `visibility_timeout` — controls when unacked messages are redelivered).

## AMQP-specific

| Setting | Default | What it does |
|---|---|---|
| `broker_login_method` | `"AMQPLAIN"` | Override for `EXTERNAL` / SSL client cert auth. |
| `broker_heartbeat` | `120.0` | RabbitMQ keepalive. |
| `broker_heartbeat_checkrate` | `2.0` | Multiplier for heartbeat checks. |

## Time and locale

```python
timezone = 'UTC'        # default; set to your TZ if you write crontabs in local time
enable_utc = True       # default; messages carry UTC ETAs
```

⚠️ If `enable_utc=False`, scheduled tasks use the worker's local time — surprising during DST.

## Imports and discovery

```python
imports = ('app.tasks', 'app.maintenance')   # explicit
include = ('app.workflows',)                  # passed to Celery(__init__)
```

In Django, `app.autodiscover_tasks()` scans `INSTALLED_APPS` for `tasks.py`.

## Inspecting current config

```python
>>> app.conf.humanize(with_defaults=False, censored=True)
>>> app.conf.table(with_defaults=False, censored=True)
```

Both hide secrets in URLs by default.

💡 Start with `broker_url`, `result_backend`, `task_serializer`, `accept_content`, `timezone`. Add `task_acks_late + task_reject_on_worker_lost + worker_prefetch_multiplier=1` once you care about reliability. Everything else is tuning. See [[Celery UG 02 - Tasks]] and [[Celery UG 05 - Workers Guide]] for the runtime behaviour these settings drive.
