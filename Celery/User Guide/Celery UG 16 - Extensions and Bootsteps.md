# UG 16 — Extensions and Bootsteps

🔑 Celery's worker is built from **bootsteps** organised into two **blueprints** (Worker and Consumer); add your own `Step` / `StartStopStep` / `ConsumerStep` to hook into the lifecycle, and register CLI commands via setuptools entry points.

## Blueprints

A bootstep is a class that defines hooks at points in the worker lifecycle. Every bootstep belongs to a blueprint. The worker has two:

### Worker blueprint

Starts first; brings up the major components. Attributes available to worker bootsteps:

- `app` — the current app.
- `hostname` — worker node name (e.g. `worker1@example.com`).
- `hub` — event loop, register callbacks here.
- `pool` — process / eventlet / gevent / thread pool.
- `timer` — for scheduling functions.
- `blueprint` — the worker `Blueprint` object.

### Consumer blueprint

Establishes the broker connection; restarts on connection loss. Attributes:

- `controller` — the parent `WorkController`.
- `connection` — current broker connection.
- `event_dispatcher` — for sending events.
- `gossip` — worker-to-worker broadcast.
- `task_consumer` — `kombu.Consumer` for task messages.
- `strategies` — task-name → strategy mapping.
- `task_buckets` — rate-limit lookups.
- `qos` — adjust prefetch_count.

## `Step` base class

Methods you may override:

- `__init__(parent, **kwargs)` — called when the `WorkController` is constructed.
- `create(parent)` — return the object whose `start`/`stop` will be called (often `self`).
- `start(parent)` — worker/consumer startup.
- `stop(parent)` — shutdown or connection restart.
- `terminate(parent)` — worker terminates (worker steps only).
- `shutdown(parent)` — consumer shutdown (consumer steps only).

## Installing bootsteps

```python
app = Celery()
app.steps['worker'].add(MyWorkerStep)
app.steps['consumer'].add(MyConsumerStep)
app.steps['consumer'].update([StepA, StepB])
```

Order is determined by the dependency graph — declare prerequisites with `Step.requires`.

## Worker bootstep example

```python
from celery import bootsteps

class ExampleWorkerStep(bootsteps.StartStopStep):
    requires = {'celery.worker.components:Pool'}

    def __init__(self, worker, **kwargs):
        print('Called when the WorkController instance is constructed')
        print('Arguments to WorkController: {0!r}'.format(kwargs))

    def create(self, worker):
        # this method can be used to delegate the action methods
        # to another object that implements ``start`` and ``stop``.
        return self

    def start(self, worker):
        print('Called when the worker is started.')

    def stop(self, worker):
        print('Called when the worker shuts down.')

    def terminate(self, worker):
        print('Called when the worker terminates')
```

## Consumer bootstep with timer

```python
from celery import bootsteps

class DeadlockDetection(bootsteps.StartStopStep):
    requires = {'celery.worker.components:Timer'}

    def __init__(self, worker, deadlock_timeout=3600):
        self.timeout = deadlock_timeout
        self.requests = []
        self.tref = None

    def start(self, worker):
        # run every 30 seconds.
        self.tref = worker.timer.call_repeatedly(
            30.0, self.detect, (worker,), priority=10,
        )

    def stop(self, worker):
        if self.tref:
            self.tref.cancel()
            self.tref = None

    def detect(self, worker):
        # update active requests
        for req in worker.active_requests:
            if req.time_start and time() - req.time_start > self.timeout:
                raise SystemExit()
```

## Custom message consumers

Embed an arbitrary Kombu consumer with `ConsumerStep`:

```python
from celery import Celery, bootsteps
from kombu import Consumer, Exchange, Queue

my_queue = Queue('custom', Exchange('custom'), 'routing_key')
app = Celery(broker='amqp://')

class MyConsumerStep(bootsteps.ConsumerStep):

    def get_consumers(self, channel):
        return [Consumer(channel,
                         queues=[my_queue],
                         callbacks=[self.handle_message],
                         accept=['json'])]

    def handle_message(self, body, message):
        print('Received message: {0!r}'.format(body))
        message.ack()

app.steps['consumer'].add(MyConsumerStep)
```

Useful when the same worker also has to listen to a non-Celery queue.

## Custom worker CLI options

```python
from celery import Celery
from click import Option

app = Celery(broker='amqp://')

app.user_options['worker'].add(Option(('--enable-my-option',),
                                      is_flag=True,
                                      help='Enable custom option.'))
```

The flag is passed into bootstep `__init__`:

```python
from celery import bootsteps

class MyBootstep(bootsteps.Step):

    def __init__(self, parent, enable_my_option=False, **options):
        super().__init__(parent, **options)
        if enable_my_option:
            party()

app.steps['worker'].add(MyBootstep)
```

`app.user_options` keys: `'preload'`, `'worker'`, `'beat'`, etc. Preload options are visible to every subcommand — see `user_preload_options` in [[Celery UG 14 - Signals]].

## Custom commands via entry points

Register a brand-new `celery <command>` via setuptools `celery.commands` entry point. In `setup.py`:

```python
setup(
    name='flower',
    entry_points={
        'celery.commands': [
           'flower = flower.command:flower',
        ],
    }
)
```

Implement with `click`:

```python
import click

@click.command()
@click.option('--port', default=8888, type=int, help='Webserver port')
@click.option('--debug', is_flag=True)
def flower(port, debug):
    print('Running our command')
```

The entry-point string is `module.path:callable`. After `pip install`, `celery flower --port 5555` works.

## When to reach for bootsteps vs signals

- **Signals** — react to lifecycle events without owning state ([[Celery UG 14 - Signals]]).
- **Bootsteps** — own a long-lived component (timer, background thread, extra Kombu consumer, healthcheck server) bound to worker/consumer lifetime.
- **Custom commands** — ship operational tooling alongside the app under the `celery` CLI umbrella.

💡 Bootsteps are the supported way to splice managed services into a worker. If you're tempted to spawn threads from inside a task or hack `celery.bin`, write a `StartStopStep` instead.
