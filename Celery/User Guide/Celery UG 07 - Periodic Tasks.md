# UG 07 — Periodic Tasks

🔑 `celery beat` is a scheduler process that publishes due tasks onto the broker; workers consume them like any other task. Run exactly one beat instance.

## Schedule entry shape

```python
from celery import Celery
from celery.schedules import crontab

app = Celery()

app.conf.beat_schedule = {
    'add-every-30-seconds': {
        'task': 'tasks.add',
        'schedule': 30.0,
        'args': (16, 16),
    },
}
app.conf.timezone = 'UTC'
```

Fields:

- `task` — registered task name (string), not import path.
- `schedule` — seconds (int/float), `timedelta`, `crontab(...)`, or `solar(...)`.
- `args`, `kwargs` — passed to the task.
- `options` — dict of `apply_async()` kwargs (`queue`, `exchange`, `priority`, ...).
- `relative` — bool; round `timedelta` to nearest second/minute/hour/day boundary.

## Programmatic registration

```python
@app.on_after_configure.connect
def setup_periodic_tasks(sender, **kwargs):
    sender.add_periodic_task(10.0, test.s('hello'), name='add every 10')
    sender.add_periodic_task(
        crontab(hour=7, minute=30, day_of_week=1),
        test.s('Happy Mondays!'),
    )

@app.task
def test(arg):
    print(arg)
```

## Crontab schedules

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    'add-every-monday-morning': {
        'task': 'tasks.add',
        'schedule': crontab(hour=7, minute=30, day_of_week=1),
        'args': (16, 16),
    },
}
```

| Expression | Meaning |
| --- | --- |
| `crontab()` | Every minute |
| `crontab(minute=0, hour=0)` | Daily at midnight |
| `crontab(minute=0, hour='*/3')` | Every 3 hours |
| `crontab(minute='*/15')` | Every 15 minutes |
| `crontab(day_of_week='sunday')` | Every minute on Sundays |
| `crontab(0, 0, day_of_month='2')` | 2nd of every month |
| `crontab(0, 0, day_of_month='2-30/2')` | Every even-numbered day |
| `crontab(0, 0, month_of_year='*/3')` | First day of each quarter |

## Solar schedules

```python
from celery.schedules import solar

app.conf.beat_schedule = {
    'add-at-melbourne-sunset': {
        'task': 'tasks.add',
        'schedule': solar('sunset', -37.81753, 144.96715),
        'args': (16, 16),
    },
}
```

Latitude `+`=N / `-`=S, longitude `+`=E / `-`=W. Events: `dawn_astronomical`, `dawn_nautical`, `dawn_civil`, `sunrise`, `solar_noon`, `sunset`, `dusk_civil`, `dusk_nautical`, `dusk_astronomical`. All use UTC; polar latitudes where the sun never rises/sets are handled.

## Timezone

Default is UTC.

```python
app.conf.timezone = 'Europe/London'
```

The default file scheduler auto-detects timezone changes and resets `celerybeat-schedule`. The Django DB scheduler does not — reset manually:

```python
from django_celery_beat.models import PeriodicTask
PeriodicTask.objects.update(last_run_at=None)
```

## Starting beat

```bash
celery -A proj beat
celery -A proj beat -s /home/celery/var/run/celerybeat-schedule
```

Embedding inside a worker (dev only):

```bash
celery -A proj worker -B
```

See [[Celery UG 06 - Daemonization]] for the production `celerybeat.service` unit.

## Custom scheduler classes

```bash
celery -A proj beat --scheduler django_celery_beat.schedulers:DatabaseScheduler
```

Django setup:

```bash
pip install django-celery-beat
```

```python
INSTALLED_APPS = (
    ...,
    'django_celery_beat',
)
```

```bash
python manage.py migrate
celery -A proj beat -l INFO --scheduler \
  django_celery_beat.schedulers:DatabaseScheduler
```

The DB scheduler lets you edit schedules at runtime via the Django admin.

## Watch out

- ⚠️ Run **exactly one** beat instance. Two beats = every periodic task fires twice. Use leader-election (`single-beat`, k8s lease, `django-celery-beat`'s DB locking) if you need HA.
- ⚠️ Tasks can overlap if a run takes longer than the period — like cron, beat doesn't serialize. Add app-level locking (Redis SETNX, DB advisory lock) when needed.
- ⚠️ `task` is the registered name, not a callable or dotted import path.

💡 Beat only schedules; workers execute. Keep schedules small and idempotent, and centralize config in `app.conf.beat_schedule` (or the DB scheduler) so it's reviewable.
