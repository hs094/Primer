# Celery Knowledge Pack

Crisp, scannable revision notes covering the Celery docs (docs.celeryq.dev/en/stable).
Source: https://docs.celeryq.dev/en/stable/

## How to use

- Each note = one concept, optimized for recall.
- Code blocks are the minimum that exercises the feature.
- 🔑 = core idea. ⚠️ = gotcha. 💡 = mental model / takeaway.
- Vocabulary check: [[Celery Glossary]].

## Getting Started

| # | Topic | Note |
|---|---|---|
| 01 | What is Celery, task queues, features | [[Celery GS 01 - Introduction to Celery]] |
| 02 | Brokers (RabbitMQ, Redis, SQS) and result backends | [[Celery GS 02 - Backends and Brokers]] |
| 03 | First app, `@task`, worker, `delay()`, config styles | [[Celery GS 03 - First Steps with Celery]] |
| 04 | Next steps — multi-module project, `app.config_from_object` | [[Celery GS 04 - Next Steps]] |
| 05 | Resources — community, docs, mailing lists, IRC | [[Celery GS 05 - Resources]] |

## User Guide

| # | Topic | Note |
|---|---|---|
| 01 | The `Celery` application object, app instances, finalization | [[Celery UG 01 - Application]] |
| 02 | Tasks — `@task`, retries, states, request context, time limits | [[Celery UG 02 - Tasks]] |
| 03 | Calling tasks — `delay`, `apply_async`, ETA, link, link_error | [[Celery UG 03 - Calling Tasks]] |
| 04 | Canvas — signatures, `group`, `chain`, `chord`, `chunks`, `map` | [[Celery UG 04 - Canvas Workflows]] |
| 05 | Workers — concurrency, pools, control commands, inspect | [[Celery UG 05 - Workers Guide]] |
| 06 | Daemonization — systemd, init scripts, supervisord | [[Celery UG 06 - Daemonization]] |
| 07 | Periodic tasks — beat, `crontab`, schedulers | [[Celery UG 07 - Periodic Tasks]] |
| 08 | Routing — queues, exchanges, `task_routes`, priorities | [[Celery UG 08 - Routing Tasks]] |
| 09 | Monitoring — Flower, events, `celery inspect/control` | [[Celery UG 09 - Monitoring and Management]] |
| 10 | Security — message signing, transport TLS, pickle warnings | [[Celery UG 10 - Security]] |
| 11 | Optimizing — prefetch, `acks_late`, pool choice, rate limits | [[Celery UG 11 - Optimizing]] |
| 12 | Debugging — rdb, eager mode, log levels, common pitfalls | [[Celery UG 12 - Debugging]] |
| 13 | Concurrency — prefork vs threads vs gevent vs eventlet | [[Celery UG 13 - Concurrency]] |
| 14 | Signals — `task_prerun`, `task_failure`, worker lifecycle | [[Celery UG 14 - Signals]] |
| 15 | Testing — `task_always_eager`, pytest fixtures, `celery_app` | [[Celery UG 15 - Testing with Celery]] |
| 16 | Extensions and bootsteps — custom worker components | [[Celery UG 16 - Extensions and Bootsteps]] |

## Configuration

| Topic | Note |
|---|---|
| All useful settings by domain (broker, results, tasks, worker, beat, security, redis, AMQP) | [[Celery Configuration and Defaults]] |

## Django

| Topic | Note |
|---|---|
| `proj/celery.py`, `CELERY_*` settings, `@shared_task`, `django-celery-results`, `django-celery-beat` | [[Celery with Django]] |

## FAQ

| Topic | Note |
|---|---|
| Common questions — PENDING, retry vs acks_late, double execution, revoke, purge | [[Celery FAQ]] |

## Reference

| Topic | Note |
|---|---|
| Glossary — ack, apply, ETA, kombu, prefetch, late ack, pidbox, … | [[Celery Glossary]] |

## Cross-links

- Configuration drives the runtime behaviour described in [[Celery UG 05 - Workers Guide]] and [[Celery UG 11 - Optimizing]].
- Workflow primitives ([[Celery UG 04 - Canvas Workflows]]) compose `signature` objects defined in [[Celery UG 03 - Calling Tasks]].
- For Django projects, start at [[Celery with Django]] and treat [[Celery Configuration and Defaults]] as the source-of-truth for setting names.
