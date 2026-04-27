# UG 01 — Application

🔑 The `Celery` app instance is the entry point: it owns config, the task registry, and the connection pool — keep it explicit, never reach for `current_app`.

## Creating an app

```python
from celery import Celery

app = Celery()
app = Celery('tasks')   # 'tasks' = main module name
```

Instantiation does four things: starts a logical clock, creates the task registry, sets itself as the current app (unless `set_as_current=False`), and calls `app.on_init()`.

The instance is thread-safe — multiple apps with different configs can coexist in one process.

## Main module name

The first positional arg sets the prefix for auto-generated task names. If unset and the module runs as `__main__`, tasks are named `__main__.add`. Set it to match what you pass to `celery -A` so worker and client agree.

## Configuration via `app.conf`

```python
app.conf.timezone = 'Europe/London'
app.conf.enable_utc = True
app.conf.update(enable_utc=True, timezone='Europe/London')
```

Precedence (high → low): runtime mutation → loaded config module → defaults.

### Loading from a module

```python
app.config_from_object('celeryconfig')   # by dotted name
import celeryconfig
app.config_from_object(celeryconfig)     # by object
```

```python
class Config:
    enable_utc = True
    timezone = 'Europe/London'

app.config_from_object(Config)
```

⚠️ `config_from_object()` resets prior settings — apply per-attribute overrides *after* it.

### From an env var

```python
import os
os.environ.setdefault('CELERY_CONFIG_MODULE', 'celeryconfig.prod')
app.config_from_envvar('CELERY_CONFIG_MODULE')
```

### Inspecting config

```python
app.conf.humanize(with_defaults=False, censored=True)
app.conf.table(with_defaults=False, censored=True)
```

Keys containing `API`, `TOKEN`, `KEY`, `SECRET`, `PASS`, `SIGNATURE`, `DATABASE` are censored.

## Lazy evaluation & finalization

Decorators don't build the task right away — they return a `PromiseProxy`:

```python
@app.task
def add(x, y):
    return x + y

type(add)              # <class 'celery.local.PromiseProxy'>
add.__evaluated__()    # False
repr(add)              # forces evaluation
```

Finalization (explicit `app.finalize()`, or implicit on first `app.tasks` access) shares tasks between apps (unless `shared=False`), evaluates pending decorators, and binds tasks to the app.

## Multiple apps

Multiple `Celery()` instances coexist freely — each has its own config and registry. Useful for splitting unrelated task domains, or for tests.

## Breaking the chain

Don't reach for `current_app` from library code — it depends on whichever app was last instantiated:

```python
# bad
from celery import current_app
class Scheduler:
    def run(self):
        app = current_app
```

Pass the app in:

```python
class Scheduler:
    def __init__(self, app):
        self.app = app
```

For backwards compatibility:

```python
from celery.app import app_or_default

class Scheduler:
    def __init__(self, app=None):
        self.app = app_or_default(app)
```

Trace surprise `current_app` use during dev:

```bash
CELERY_TRACE_APP=1 celery worker -l INFO
```

## Abstract task base classes

Every `@app.task` produces a class inheriting from `app.Task`. Override per task with `base=`:

```python
from celery import Task

class DebugTask(Task):
    def __call__(self, *args, **kwargs):
        print(f'TASK STARTING: {self.name}[{self.request.id}]')
        return self.run(*args, **kwargs)

@app.task(base=DebugTask)
def add(x, y):
    return x + y
```

⚠️ In a custom `__call__`, call `self.run(...)` — *not* `super().__call__(...)`.

Change the default app-wide base class:

```python
app.Task = MyBaseTask
```

💡 One app, one config source of truth, no `current_app` in business logic. See [[Celery UG 02 - Tasks]] for what `@app.task` actually produces.
