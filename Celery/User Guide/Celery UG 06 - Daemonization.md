# UG 06 — Daemonization

🔑 In production, run workers and beat under a process supervisor (systemd preferred). Config lives in `/etc/conf.d/celery`; the unit files just shell out to `celery multi` and `celery beat`.

## Pick a supervisor

- **systemd** — modern Linux default. Verify: `systemctl --version`.
- **generic init.d** — `extra/generic-init.d/` for systems without systemd (still works via `systemd-sysv` shim).
- **supervisord** — see `extra/supervisord/`.
- **launchd** — macOS, see `extra/macOS/`.

## Never run as root

Workers can execute arbitrary code from serialized messages. Celery refuses to start as root by default; override only with `C_FORCE_ROOT=1`. Without it, the worker silently exits.

## /etc/conf.d/celery (shared environment file)

```bash
CELERYD_NODES="w1"
CELERY_BIN="/usr/local/bin/celery"
CELERY_APP="proj"
CELERYD_MULTI="multi"
CELERYD_OPTS="--time-limit=300 --concurrency=8"
CELERYD_PID_FILE="/var/run/celery/%n.pid"
CELERYD_LOG_FILE="/var/log/celery/%n%I.log"
CELERYD_LOG_LEVEL="INFO"
CELERYBEAT_PID_FILE="/var/run/celery/beat.pid"
CELERYBEAT_LOG_FILE="/var/log/celery/beat.log"
```

## systemd: celery.service

```ini
[Unit]
Description=Celery Service
After=network.target

[Service]
Type=forking
User=celery
Group=celery
EnvironmentFile=/etc/conf.d/celery
WorkingDirectory=/opt/celery
ExecStart=/bin/sh -c '${CELERY_BIN} -A $CELERY_APP multi start $CELERYD_NODES \
    --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} \
    --loglevel="${CELERYD_LOG_LEVEL}" $CELERYD_OPTS'
ExecStop=/bin/sh -c '${CELERY_BIN} multi stopwait $CELERYD_NODES \
    --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} \
    --loglevel="${CELERYD_LOG_LEVEL}"'
ExecReload=/bin/sh -c '${CELERY_BIN} -A $CELERY_APP multi restart $CELERYD_NODES \
    --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} \
    --loglevel="${CELERYD_LOG_LEVEL}" $CELERYD_OPTS'
Restart=always

[Install]
WantedBy=multi-user.target
```

`Type=forking` because `celery multi` daemonizes itself. `stopwait` blocks until tasks drain.

## systemd: celerybeat.service

```ini
[Unit]
Description=Celery Beat Service
After=network.target

[Service]
Type=simple
User=celery
Group=celery
EnvironmentFile=/etc/conf.d/celery
WorkingDirectory=/opt/celery
ExecStart=/bin/sh -c '${CELERY_BIN} -A ${CELERY_APP} beat  \
    --pidfile=${CELERYBEAT_PID_FILE} \
    --logfile=${CELERYBEAT_LOG_FILE} --loglevel=${CELERYD_LOG_LEVEL}'
Restart=always

[Install]
WantedBy=multi-user.target
```

Beat is `Type=simple` — it stays in the foreground.

## Enable

```bash
systemctl daemon-reload
systemctl enable celery.service
systemctl enable celerybeat.service
```

## tmpfiles.d (run/log dirs survive reboots)

`/etc/tmpfiles.d/celery.conf`:

```bash
d /run/celery 0755 celery celery -
d /var/log/celery 0755 celery celery -
```

## Generic init.d (legacy)

```bash
/etc/init.d/celeryd {start|stop|restart|status}
/etc/init.d/celerybeat {start|stop|restart}
```

Configured via `/etc/default/celeryd`:

```bash
CELERYD_NODES="worker1"
CELERY_BIN="/usr/local/bin/celery"
CELERY_APP="proj"
CELERYD_CHDIR="/opt/Myproject/"
CELERYD_OPTS="--time-limit=300 --concurrency=8"
CELERYD_LOG_FILE="/var/log/celery/%n%I.log"
CELERYD_PID_FILE="/var/run/celery/%n.pid"
CELERYD_USER="celery"
CELERYD_GROUP="celery"
CELERY_CREATE_DIRS=1
```

Key vars: `CELERYD_NODES`, `CELERY_BIN`, `CELERY_APP`, `CELERYD_CHDIR`, `CELERYD_OPTS`, `CELERYD_LOG_FILE`, `CELERYD_PID_FILE`, `CELERYD_USER`, `CELERYD_GROUP`, `CELERY_CREATE_DIRS`. Beat-specific: `CELERYBEAT_OPTS`, `CELERYBEAT_PID_FILE`, `CELERYBEAT_LOG_FILE`, `CELERYBEAT_USER`, `CELERYBEAT_GROUP`.

## Unprivileged alternative

Skip init scripts entirely:

```bash
celery -A proj multi start worker1 \
    --pidfile="$HOME/run/celery/%n.pid" \
    --logfile="$HOME/log/celery/%n%I.log"
```

## Django

Set `DJANGO_SETTINGS_MODULE` in the module that defines the Celery app instance. See [[Celery UG 02 - Tasks]] for app layout.

## Troubleshooting

Run the init script verbose:

```bash
sh -x /etc/init.d/celeryd start
```

Skip the daemonization fork to see startup errors:

```bash
C_FAKEFORK=1 sh -x /etc/init.d/celeryd start
```

## Watch out

- ⚠️ Multiple beat instances = duplicate runs. See [[Celery UG 07 - Periodic Tasks]].
- ⚠️ `Type=forking` requires the pidfile to exist after `ExecStart` returns — `celery multi` writes it.
- ⚠️ Don't set `CELERYBEAT_OPTS` to a stale `--schedule=` path on redeploy if the schedule format changed.

💡 Keep all knobs in `/etc/conf.d/celery`; the unit files are just plumbing.
