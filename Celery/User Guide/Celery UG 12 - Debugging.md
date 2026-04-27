# UG 12 — Debugging

🔑 `celery.contrib.rdb` is `pdb` over a socket — drop a `set_trace()` in a task, telnet into the worker, and you have an interactive shell on the live process.

## Why a remote debugger

Workers usually run detached: daemonized, in a container, or on another host. There is no terminal to attach `pdb` to. `rdb` is described in the docs as "an extended version of pdb that enables remote debugging of processes that doesn't have terminal access." It opens a TCP socket on a breakpoint and serves a pdb session over it.

## Basic usage

```python
from celery import task
from celery.contrib import rdb

@task()
def add(x, y):
    result = x + y
    rdb.set_trace()  # <- set break-point
    return result
```

When the task hits `set_trace()`, the worker logs the listening port and blocks until something connects. All other slots on that worker continue executing unless they too hit a breakpoint.

## Connecting

```bash
$ telnet localhost 6900
Connected to localhost.
Escape character is '^]'.
> /opt/devel/demoapp/tasks.py(128)add()
-> return result
(Pdb)
```

From here you have a normal pdb session: `n`, `s`, `c`, `p`, `pp`, `l`, `w`, `bt`, `!expr`. Hitting `c` (continue) closes the socket and the task returns normally.

## Port allocation

`rdb` picks the first free port starting at **6900** by default. With concurrency > 1 and multiple breakpoints, each hit gets its own port — read the worker log to find the right one.

```bash
export CELERY_RDB_PORT=6900    # change base port
```

## Remote access

By default the listener binds to localhost only. To debug a worker on another host:

```bash
export CELERY_RDB_HOST=0.0.0.0
```

⚠️ This opens an unauthenticated Python shell on a TCP port. Bind it to an internal interface, gate it with a firewall / SSH tunnel, and never enable it in production-facing networks.

Safer pattern — leave the host as `localhost` and tunnel:

```bash
ssh -L 6900:localhost:6900 worker-host
telnet localhost 6900
```

## Triggering a breakpoint without redeploying

If you cannot edit code (or the bug only reproduces under specific live state), arm signal-driven traps at worker start:

```bash
CELERY_RDBSIG=1 celery -A proj worker -l INFO
```

Then send `SIGUSR2` to a worker process to drop *that* process into `rdb` at its current frame:

```bash
kill -USR2 <pid>
```

`ps -ef | grep celery` to find the right pid (usually a child process under the prefork master). The signal handler calls `rdb.set_trace(frame)` on the running stack, so you land wherever execution happened to be.

## Workflow tips

- Run a single worker with `-c 1` while debugging — reduces the chance another slot races into a second breakpoint.
- Use `--pool=solo` to avoid `fork()` complications when stepping into C extensions or libraries with thread-local state.
- Combine with `celery -A proj events --dump` ([[Celery UG 09 - Monitoring and Management]]) to correlate the breakpoint with the cluster-level event stream.
- For "why is this task stuck?" without code changes, prefer `CELERY_RDBSIG=1` + `kill -USR2` over redeploying with a hard-coded `set_trace`.

## When rdb is wrong

- **Reproducible logic bugs** — a unit test with `pytest` and `app.conf.task_always_eager = True` is faster.
- **Worker hangs on shutdown** — use `py-spy dump --pid <pid>` to read the stack without holding execution.
- **Performance** — `cProfile` or `py-spy record`, not pdb.

See [[Celery UG 02 - Tasks]] for `task_always_eager` and [[Celery UG 06 - Workers]] for sending `TERM`/`USR1`/`USR2` signals to workers.

💡 `rdb.set_trace()` for known points, `CELERY_RDBSIG=1` + `SIGUSR2` for live processes you cannot redeploy — and tunnel the socket, do not expose it.
