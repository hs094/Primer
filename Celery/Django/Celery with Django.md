# Celery with Django

🔑 In Django, the Celery app lives in `proj/celery.py`, settings live in `settings.py` under a `CELERY_` namespace, and tasks live in each app's `tasks.py` decorated with `@shared_task` so they don't import the app instance.

## File layout

```
proj/
├── manage.py
├── proj/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── celery.py        ← the Celery app
└── demoapp/
    ├── models.py
    ├── views.py
    └── tasks.py         ← @shared_task functions
```

## `proj/celery.py`

```python
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'proj.settings')

app = Celery('proj')

# Pull config from Django settings, only keys prefixed CELERY_.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Discover tasks.py inside every INSTALLED_APP.
app.autodiscover_tasks()


@app.task(bind=True, ignore_result=True)
def debug_task(self):
    print(f'Request: {self.request!r}')
```

The string reference `'django.conf:settings'` matters — the worker doesn't have to pickle the settings object to child processes.

## `proj/__init__.py`

```python
from .celery import app as celery_app

__all__ = ('celery_app',)
```

This guarantees the app is loaded when Django starts so `@shared_task` can bind to it.

## `settings.py` — the `CELERY_` namespace

With `namespace='CELERY'` every setting from [[Celery Configuration and Defaults]] becomes `CELERY_<UPPER_NAME>`:

```python
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/1'

CELERY_TIMEZONE = 'Australia/Tasmania'
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 30 * 60

CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
```

So `task_always_eager` → `CELERY_TASK_ALWAYS_EAGER`, `broker_url` → `CELERY_BROKER_URL`, etc.

## `@shared_task` vs `@app.task`

In a reusable Django app you cannot import `proj.celery.app` — that creates a circular dependency between the project and the app. Use `@shared_task` instead:

```python
# demoapp/tasks.py
from celery import shared_task
from demoapp.models import Widget


@shared_task
def add(x, y):
    return x + y


@shared_task
def count_widgets():
    return Widget.objects.count()
```

`@shared_task` binds to whichever Celery app is current at the time the task is called — exactly what `proj/__init__.py` sets up.

Use `@app.task` only inside `proj/celery.py` itself (e.g. for `debug_task`), where you already have the app handle.

## Results — `django-celery-results`

Stores results in the Django ORM (or cache) instead of Redis/AMQP, so they're queryable from the admin.

```bash
pip install django-celery-results
```

```python
# settings.py
INSTALLED_APPS = [
    ...,
    'django_celery_results',
]

CELERY_RESULT_BACKEND = 'django-db'
# or:
# CELERY_RESULT_BACKEND = 'django-cache'
```

```bash
python manage.py migrate django_celery_results
```

## Periodic tasks — `django-celery-beat`

Stores `beat_schedule` in the database so you can edit it from the admin without restarting beat.

```bash
pip install django-celery-beat
```

```python
INSTALLED_APPS = [
    ...,
    'django_celery_beat',
]

CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'
```

```bash
python manage.py migrate django_celery_beat
```

Run beat alongside the worker:

```bash
celery -A proj beat -l INFO
```

## Running the worker

```bash
celery -A proj worker -l INFO
```

In production run worker (and beat) under systemd / supervisord — never `manage.py` and never in the same process as the web server.

## Calling tasks from views

```python
# demoapp/views.py
from django.http import JsonResponse
from .tasks import add


def trigger(request):
    result = add.delay(2, 3)
    return JsonResponse({'task_id': result.id})
```

⚠️ Don't call `result.get()` inside a request handler — that turns the async dispatch into a synchronous call and blocks the request worker.

## Testing

Set `CELERY_TASK_ALWAYS_EAGER = True` in test settings so tasks execute inline. Pair with `CELERY_TASK_EAGER_PROPAGATES = True` to surface exceptions. See [[Celery UG 14 - Testing with Celery]].

💡 The mental model: `proj/celery.py` is the only file that touches the `Celery()` instance. Everything else uses `@shared_task` and `settings.CELERY_*`. Treat the worker as a peer process to `runserver`, not a Django subsystem.
