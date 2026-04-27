# UG 08 — Routing Tasks

🔑 Tasks publish to an **exchange** with a **routing key**; the broker delivers to **queues** bound to that exchange. `task_routes` is the cheap path; `task_queues` + custom routers is the full AMQP version.

## Simplest: route by name

```python
task_routes = {'feed.tasks.import_feed': {'queue': 'feeds'}}
```

`task_create_missing_queues` (default `True`) auto-declares unknown queues. Other tasks fall through to `task_default_queue` (default `celery`).

Glob:

```python
app.conf.task_routes = {'feed.tasks.*': {'queue': 'feeds'}}
```

Priority-ordered (list, regex allowed):

```python
import re
task_routes = ([
    ('feed.tasks.*', {'queue': 'feeds'}),
    ('web.tasks.*',  {'queue': 'web'}),
    (re.compile(r'(video|image)\.tasks\..*'), {'queue': 'media'}),
],)
```

Start workers per queue:

```bash
celery -A proj worker -Q feeds
celery -A proj worker -Q feeds,celery
```

## Default queue / exchange

```python
app.conf.task_default_queue = 'default'
app.conf.task_default_exchange = 'tasks'
app.conf.task_default_exchange_type = 'topic'
app.conf.task_default_routing_key = 'task.default'
```

Auto-created queues use `{'exchange': name, 'exchange_type': 'direct', 'routing_key': name}`. **Redis/SQS:** queue name must match exchange and routing key.

## Manual queues with kombu

```python
from kombu import Queue

app.conf.task_default_queue = 'default'
app.conf.task_queues = (
    Queue('default',    routing_key='task.#'),
    Queue('feed_tasks', routing_key='feed.#'),
)
app.conf.task_default_exchange = 'tasks'
app.conf.task_default_exchange_type = 'topic'
app.conf.task_default_routing_key = 'task.default'

task_routes = {
    'feeds.tasks.import_feed': {
        'queue': 'feed_tasks',
        'routing_key': 'feed.import',
    },
}
```

Override per-call:

```python
from feeds.tasks import import_feed
import_feed.apply_async(
    args=['http://cnn.com/rss'],
    queue='feed_tasks',
    routing_key='feed.import',
)
```

## Multiple exchanges & bindings

```python
from kombu import Exchange, Queue, binding

app.conf.task_queues = (
    Queue('feed_tasks',    routing_key='feed.#'),
    Queue('regular_tasks', routing_key='task.#'),
    Queue('image_tasks',
          exchange=Exchange('mediatasks', type='direct'),
          routing_key='image.compress'),
)

media_exchange = Exchange('media', type='direct')
CELERY_QUEUES = (
    Queue('media', [
        binding(media_exchange, routing_key='media.video'),
        binding(media_exchange, routing_key='media.image'),
    ]),
)
```

## AMQP primer

Producer → **exchange** → (routing key + bindings) → **queue** → consumer ack → message deleted.

Exchange types:

- **direct** — routing key must match exactly.
- **topic** — dotted patterns; `*` matches one word, `#` matches zero+ words. `*.news` matches `usa.news`; `usa.#` matches `usa.weather` and `usa.sports.nba`.
- **fanout** — broadcast to every bound queue, ignore routing key.

Message envelope:

```python
{'task': 'myapp.tasks.add',
 'id': '54086c5e-6193-4575-8308-dbab76798756',
 'args': [4, 4],
 'kwargs': {}}
```

## Custom router function

```python
def route_task(name, args, kwargs, options, task=None, **kw):
    if name == 'myapp.tasks.compress_video':
        return {'exchange': 'video',
                'exchange_type': 'topic',
                'routing_key': 'video.compress'}
```

Install:

```python
task_routes = (route_task,)
# or by dotted path
task_routes = ('myapp.routers.route_task',)
```

## Priority

RabbitMQ (per-queue):

```python
from kombu import Exchange, Queue

app.conf.task_queues = [
    Queue('tasks', Exchange('tasks'), routing_key='tasks',
          queue_arguments={'x-max-priority': 10}),
]
app.conf.task_queue_max_priority = 10
app.conf.task_default_priority = 5
```

Redis (transport-level, **0 = highest**):

```python
app.conf.broker_transport_options = {
    'queue_order_strategy': 'priority',
}
```

Per-task:

```python
task.apply_async(priority=0)
```

## Broadcast (fanout)

```python
from kombu.common import Broadcast

app.conf.task_queues = (Broadcast('broadcast_tasks'),)
app.conf.task_routes = {
    'tasks.reload_cache': {
        'queue': 'broadcast_tasks',
        'exchange': 'broadcast_tasks',
    },
}
```

Every worker subscribed to `broadcast_tasks` runs the task — useful for cache invalidation. Combine with beat:

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    'test-task': {
        'task': 'tasks.reload_cache',
        'schedule': crontab(minute=0, hour='*/3'),
        'options': {'exchange': 'broadcast_tasks'},
    },
}
```

See [[Celery UG 07 - Periodic Tasks]].

## Resolution order

1. kwargs to `apply_async()`
2. Task routing attributes
3. `task_routes`

## CLI AMQP shell

```bash
celery -A proj amqp
1> exchange.declare testexchange direct
2> queue.declare testqueue
3> queue.bind testqueue testexchange testkey
4> basic.publish 'This is a message!' testexchange testkey
5> basic.get testqueue
6> basic.ack 1
7> queue.delete testqueue
8> exchange.delete testexchange
```

## Watch out

- ⚠️ Redis/SQS treat queue name = exchange = routing key. Topic-style patterns won't work; switch broker if you need them.
- ⚠️ `Broadcast` queues are non-durable and unique per worker — don't use them for work that must survive restarts.
- ⚠️ Redis priority is **inverted** vs RabbitMQ (0 highest, not lowest).

💡 Start with `task_routes` + `-Q queuename`. Reach for `Queue`/`Exchange`/router functions only when you actually need topic patterns or multiple bindings.
