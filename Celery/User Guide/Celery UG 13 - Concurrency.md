# UG 13 — Concurrency

🔑 Pool choice is the single biggest perf knob: `prefork` for CPU-bound (default), `eventlet`/`gevent` for I/O-bound (thousands of green threads), `threads` when you want OS threads, `solo` to skip pooling entirely.

## Pool types

Five pools selectable via `--pool` / `-P`:

- **prefork** — default. Multiple OS processes via `billiard`. Best for CPU-bound work and the safest default.
- **eventlet** — greenlets via `eventlet`. Best for I/O-bound (HTTP, slow sockets).
- **gevent** — greenlets via `gevent`. Same niche as eventlet.
- **threads** — OS threads via `concurrent.futures.ThreadPoolExecutor`. Requires the stdlib module.
- **solo** — runs tasks **sequentially in the main thread**. No pool, no concurrency.
- **custom** — set via env var to point at your own pool implementation.

⚠️ Switching away from `prefork` **silently disables** features like `soft_timeout` and `max_tasks_per_child`.

## CLI flags

```bash
celery -A proj worker -P prefork -c 8
celery -A proj worker -P eventlet -c 1000
celery -A proj worker -P gevent -c 500
celery -A proj worker -P threads -c 20
celery -A proj worker -P solo
```

- `-P, --pool` — pool implementation.
- `-c, --concurrency` — number of worker processes / greenlets / threads. Default for prefork is the number of CPUs.

## When each pool fits

| Workload | Pool |
|---|---|
| CPU-heavy (encoding, ML inference, parsing) | `prefork` |
| Many slow HTTP / DB / socket calls | `eventlet` or `gevent` |
| Mixed, blocking C extensions you can't monkey-patch | `prefork` (or `threads`) |
| Debugging / single-task sanity checks | `solo` |
| Stdlib-only deployments where you want true threads | `threads` |

## Eventlet — the I/O pool

Eventlet swaps blocking I/O for non-blocking via `epoll`/`libevent` while keeping a synchronous-looking programming style. The docs note it can spawn "hundreds, or thousands of green threads" — informally, **hundreds of feeds/sec on eventlet vs ~14s for 100 feeds on prefork** for asynchronous HTTP work.

```bash
celery -A proj worker -P eventlet -c 1000
```

### Caveats

- **CPU-bound tasks are unsuitable** — one busy greenlet starves the rest.
- **C extensions may not be monkey-patched.** `pylibmc` doesn't cooperate; `psycopg2` does.
- **Don't block the event loop.** Any sync `time.sleep`, blocking C call, or tight CPU loop freezes every greenlet on the worker.

### Mixed-fleet pattern

The docs recommend running both pools side by side and routing tasks by compatibility — eventlet workers consume an `io` queue, prefork workers consume a `cpu` queue.

## Monkey patching

Eventlet/gevent require the stdlib to be patched so `socket`, `ssl`, `time`, `select`, etc. become cooperative. Celery's worker does this automatically when started with `-P eventlet` / `-P gevent`. If you import `requests`, `urllib3`, etc. at module load and they cache references **before** patching, those calls stay blocking — keep heavy imports inside the task body or ensure patching happens at process start.

## Solo

```bash
celery -A proj worker -P solo
```

Executes one task at a time in the main thread. Useful for debugging, integration tests, or environments where forking is unsafe (some macOS configs, restricted containers).

## Threads

```bash
celery -A proj worker -P threads -c 20
```

Backed by `concurrent.futures.ThreadPoolExecutor`. Sits between solo and prefork: shares process memory like greenlets, but uses real OS threads — so the GIL still bottlenecks CPU-bound work.

## Concurrency vs prefetch

Pool size (`-c`) is independent from broker prefetch (`worker_prefetch_multiplier`). See [[Celery UG 05 - Workers Guide]] for prefetch tuning and [[Celery UG 03 - Routing Tasks]] for routing per-pool queues.

## Related signals

The eventlet pool fires its own lifecycle signals: `eventlet_pool_started`, `eventlet_pool_preshutdown`, `eventlet_pool_postshutdown`, `eventlet_pool_apply`. See [[Celery UG 14 - Signals]].

💡 Default to `prefork`. Reach for `eventlet`/`gevent` only when the bottleneck is **measured** I/O wait — and split fleets so CPU tasks never land on the I/O pool.
