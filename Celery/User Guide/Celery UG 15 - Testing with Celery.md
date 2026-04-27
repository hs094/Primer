# UG 15 — Testing with Celery

🔑 Use the `celery.contrib.pytest` plugin: `celery_app` + `celery_worker` give you a real worker in a thread for honest integration tests, and `celery_config` / `celery_parameters` let you override config per session — `task_always_eager` is **not** a substitute.

## Enabling the pytest plugin

The plugin ships disabled. Enable it via one of:

1. Install with the extra:

```bash
pip install celery[pytest]
```

2. Set an env var:

```bash
PYTEST_PLUGINS=celery.contrib.pytest
```

3. Register in `conftest.py`:

```python
pytest_plugins = ("celery.contrib.pytest", )
```

## Function-scoped fixtures

`celery_app` — a Celery app for the test. `celery_worker` — a live worker started in a separate thread that consumes tasks from the broker for real:

```python
def test_create_task(celery_app, celery_worker):
    @celery_app.task
    def mul(x, y):
        return x * y
    celery_worker.reload()
    assert mul.delay(4, 4).get(timeout=10) == 16
```

`celery_worker.reload()` is required after registering tasks at runtime so the worker picks them up.

## Session-scoped fixtures

Override `celery_config` to point at the broker and result backend used by the test session:

```python
@pytest.fixture(scope='session')
def celery_config():
    return {
        'broker_url': 'amqp://',
        'result_backend': 'redis://'
    }
```

Other session fixtures:

- `celery_parameters` — kwargs passed to `Celery.__init__`.
- `celery_worker_parameters` — kwargs for the embedded `WorkController` (queues, etc.).
- `celery_includes` — extra modules to import on the embedded worker.
- `celery_worker_pool` — pool implementation for the embedded worker (e.g. `'prefork'`, `'solo'`).
- `celery_enable_logging` — opt logging on inside embedded workers.
- `celery_session_app` — Celery app cached for the whole session.
- `celery_session_worker` — live worker that persists across tests.
- `use_celery_app_trap` — when on, accessing the **default app** raises, to keep tests honest.

## ⚠️ Eager mode is not a unit-test strategy

> "The eager mode enabled by the `task_always_eager` setting is by definition not suitable for unit tests."

`task_always_eager` makes `.delay()` / `.apply_async()` execute the task synchronously **in the caller**. It only emulates worker behaviour and produces real discrepancies vs production: serialization round-trips, retries via the broker, prefetch, soft timeouts, signals firing in the worker process, and result-backend behaviour are all skipped or different. Use a live `celery_worker` fixture for integration tests instead.

## Mocking apply_async for unit tests

For pure unit tests of code that **calls** a task, mock the call rather than running it. The dispatch side is small and stable:

```python
from unittest.mock import patch

def test_handler_enqueues_email():
    with patch('proj.tasks.send_email.apply_async') as m:
        handler(user_id=42)
    m.assert_called_once_with(args=(42,), countdown=10)
```

This isolates the unit from broker, worker, and serializer concerns. Reserve `celery_worker` for tests that need to assert end-to-end behaviour.

## Choosing scopes

- **Function-scoped** (`celery_app`, `celery_worker`) — clean per-test app, slower; each test gets a fresh worker thread.
- **Session-scoped** (`celery_session_app`, `celery_session_worker`) — start the worker once for the whole run; faster but tests must not mutate global state.

## Pool choice for tests

Set `celery_worker_pool` to `'solo'` for deterministic single-threaded execution (easier debugging, no fork on macOS), or `'prefork'` to match production. See [[Celery UG 13 - Concurrency]].

## Putting it together

```python
# conftest.py
import pytest

pytest_plugins = ("celery.contrib.pytest",)


@pytest.fixture(scope='session')
def celery_config():
    return {
        'broker_url': 'memory://',
        'result_backend': 'cache+memory://',
    }


@pytest.fixture(scope='session')
def celery_includes():
    return ['proj.tasks']
```

```python
# test_tasks.py
def test_add(celery_app, celery_worker):
    from proj.tasks import add
    assert add.delay(2, 3).get(timeout=5) == 5
```

## What the plugin does not give you

No assertions on signals fired, no built-in mocks for `chord` / `group` orchestration, no fake broker that records messages — for those, hand-roll fixtures or use `unittest.mock` against `Task.apply_async` and the `celery.signals` module.

💡 For unit tests, mock `apply_async`. For integration tests, run a real `celery_worker` against an in-memory or test broker. Never lean on `task_always_eager`.
