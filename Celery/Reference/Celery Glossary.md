# Celery Glossary

🔑 Celery has a small, precise vocabulary — `apply` ≠ `call`, `ack` ≠ `late ack`, `prefetch count` ≠ `prefetch multiplier`. Get these right and the docs stop being confusing.

## A

**ack** — Short for *acknowledged*.

**acknowledged** — Workers acknowledge messages to signify that a message has been handled. Failing to acknowledge a message will cause the message to be redelivered.

**apply** — Originally a synonym for *call*, but now used to signify that a function is executed by the **current process** (i.e. synchronously, in the caller). Compare `task.apply()` (runs locally) with `task.apply_async()` (sends to the broker).

## B

**billiard** — Fork of the Python `multiprocessing` library containing improvements required by Celery. Backs the default `prefork` worker pool.

## C

**calling** — Sends a task message so that the task function is executed by a worker. Done via `task.delay(...)` or `task.apply_async(...)`.

**cipater** — Code-name of Celery release 3.1, after a song by Autechre.

**context** — The runtime metadata attached to a task: its id, arguments, retry count, the queue it was delivered to, the worker hostname, and so on. Accessed via `self.request` when the task is `@task(bind=True)`.

## E

**early ack** — Short for *early acknowledgment*.

**early acknowledgment** — Task is acknowledged just-in-time before being executed, meaning the task **won't be redelivered** to another worker if the machine loses power. This is the **default** behaviour. Trade reliability for throughput.

**ETA** — *Estimated Time of Arrival*. Used as the term for a delayed message that should not be processed until the specified ETA time. Set via `apply_async(eta=...)` or `apply_async(countdown=...)`.

**executing** — What workers do with task requests.

## I

**idempotent** — A function that can be called multiple times without changing the result. The property you need before turning on `acks_late` or any retry logic.

## K

**kombu** — Python messaging library used by Celery to send and receive messages. Provides the broker abstractions (RabbitMQ, Redis, SQS, …).

## L

**late ack** — Short for *late acknowledgment*.

**late acknowledgment** — Task is acknowledged **after** execution (whether successful or raising an error). Pair with `task_reject_on_worker_lost=True` to recover tasks from crashed workers. Requires idempotent tasks.

## N

**nullipotent** — A function with the same effect and result whether called zero or many times — i.e. side-effect free. Stronger than idempotent.

## P

**pidbox** — A process mailbox, used to implement remote control commands (`celery control ...`, `celery inspect ...`).

**prefetch count** — Maximum number of unacknowledged messages a consumer can hold at once. The fundamental knob for fairness vs. throughput.

**prefetch multiplier** — The configured prefetch count is computed as `worker_prefetch_multiplier × pool_slots`. Set the multiplier to `1` for long or uneven tasks so workers grab one job at a time.

## R

**reentrant** — A function that can be interrupted mid-execution and safely called again before the first call completes.

**request** — Task messages, as decoded inside the worker. The `self.request` object on a bound task is exactly this.

💡 The two pairs to keep straight: *apply* is local / synchronous, *calling* goes through the broker; *early ack* is fast and lossy on crash, *late ack* is reliable but needs idempotence. See [[Celery UG 02 - Tasks]] and [[Celery UG 05 - Workers Guide]].
