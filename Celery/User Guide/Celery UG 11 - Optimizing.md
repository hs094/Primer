# UG 11 — Optimizing

🔑 Most Celery throughput problems are not Celery — they are the prefetch multiplier fighting your task duration, the wrong pool for the workload, or a result backend on the hot path.

## Ensure the queue can drain

> If a task takes 10 minutes to complete, and there are 10 new tasks coming in every minute, the queue will never be empty.

Monitor queue depth (Munin, Flower, `rabbitmqctl list_queues`, `redis-cli llen`) and alert on a drift trend, not an absolute threshold. See [[Celery UG 09 - Monitoring and Management]].

## Broker connection pool

Connections are pooled by default since 2.5. Tune with:

```python
app.conf.broker_pool_limit = 10   # default; set to None to disable pooling
```

Size to the number of threads / green-threads that publish concurrently. Disable (`None`) only when you have many short-lived publishers and connection churn is acceptable — usually you want pooling.

## Transient queues

For non-critical work, switch off broker disk persistence:

```python
from kombu import Queue
app.conf.task_queues = (
    Queue('transient', routing_key='transient', durable=False,
          queue_arguments={'x-message-ttl': 60_000}),
)
# delivery_mode=1 (transient) vs default delivery_mode=2 (persistent)
```

Route via `queue='transient'` on the call or `task_routes`. Trades durability for ~order-of-magnitude latency on RabbitMQ.

## Result backend

The backend sits on the hot path — every result write blocks the worker slot until ack. Pick by trade-off:

- **`rpc://`** — replies over AMQP, per-client queues. Cheap, no extra service, but results are not persistent and a single `AsyncResult` can only be fetched once.
- **`redis://`** — fast, persistent for a TTL window, supports `result_expires`. Default for most production setups; watch memory if you store large payloads.
- **Database (`db+postgresql://`, etc.)** — durable, queryable, slowest. Use when results are part of the domain model, not when you just need `.get()`.
- **`disabled`/`ignore_result=True`** — fastest. Set per-task with `@app.task(ignore_result=True)` when nothing reads the result.

```python
app.conf.update(
    result_backend='redis://localhost/0',
    result_expires=3600,
    task_ignore_result=True,   # global default; opt back in per-task
)
```

## Worker concurrency

```bash
celery -A proj worker -c 8                  # 8 prefork processes
celery -A proj worker --pool=gevent -c 500  # 500 greenlets
```

```python
app.conf.worker_concurrency = 8
```

- **prefork** (default) — one OS process per slot, immune to GIL, right for CPU-bound or blocking C extensions.
- **gevent / eventlet** — green threads, cheap context switch, hundreds-to-thousands of concurrent slots. Use for I/O-bound work (HTTP, DB) where most time is spent waiting. Requires monkey-patching; CPU-bound code starves the loop.
- **solo** — single-threaded, useful for debugging or environments without `fork()`.
- **threads** — OS threads; bound by GIL for Python code.

Rule of thumb: prefork ≈ CPU count for compute, gevent in the hundreds for fan-out HTTP.

## Prefetch — the gotcha

```python
app.conf.worker_prefetch_multiplier = 4   # default
```

Effective prefetch = `worker_prefetch_multiplier * concurrency`. The default (4) is tuned for many short tasks: the worker reserves a buffer so a slot never idles between messages.

⚠️ For long-running tasks this is a trap. If one worker prefetches 4 × 8 = 32 hour-long tasks, those messages sit in its private buffer for the next 32 hours while idle workers in the cluster see nothing to do. Set:

```python
app.conf.worker_prefetch_multiplier = 1
app.conf.task_acks_late = True
```

Now the worker holds exactly one task per slot, ack happens after success, and a crash redelivers — which is the right shape for non-trivial work.

`task_acks_late = True` only makes sense if your tasks are idempotent. See [[Celery UG 02 - Tasks]].

## Memory recycling

```python
app.conf.worker_max_tasks_per_child = 1000      # restart child after N tasks
app.conf.worker_max_memory_per_child = 200_000  # KiB; restart over threshold
```

Cheap insurance against slow leaks. Setting them too low (e.g. 1 task per child) inverts the tradeoff — fork overhead dominates.

## Skip cluster chatter

Each worker by default exchanges presence/state with every other worker. On large fleets, the gossip + mingle channels are non-trivial overhead and rarely useful:

```bash
celery -A proj worker --without-gossip --without-mingle --without-heartbeat
```

- `--without-gossip` — skip worker-to-worker event subscription.
- `--without-mingle` — skip the startup sync that fetches logical clock + revoked-tasks set from peers.
- `--without-heartbeat` — disable broker connection heartbeats (only safe with a healthy connection layer).

Useful for autoscaling fleets where workers come and go constantly.

See [[Celery UG 06 - Workers]] for pool internals and [[Celery UG 10 - Security]] for `broker_pool_limit` × TLS interactions.

💡 Default `worker_prefetch_multiplier=4` is for fast tasks; the moment a task can run for minutes, drop it to `1` with `task_acks_late=True` and pick a pool that matches CPU vs I/O.
