# EX 03 — Queues

🔑 **Key insight:** queues carry bytes, not objects — `model_dump_json()` on the way in and `model_validate_json()` on the way out turns any broker into a typed pipeline with validation at both ends.

## The pattern

Producer serializes a Pydantic model to JSON and pushes; consumer pops and revalidates. The schema travels with the payload, but the **contract** is enforced on receipt — never trust that the producer sent something well-formed. Same `User` model across all three brokers below:

```python
from pydantic import BaseModel, EmailStr

class User(BaseModel):
    id: int
    name: str
    email: EmailStr
```

See [[Pydantic CO 01 - Models]] and [[Pydantic CO 09 - Serialization]] for the dump/validate roundtrip.

## Redis list as a queue

`rpush` to enqueue, `lpop` to dequeue. Validation happens *after* the pop:

```python
import redis

from pydantic import BaseModel, EmailStr


class User(BaseModel):
    id: int
    name: str
    email: EmailStr


r = redis.Redis(host='localhost', port=6379, db=0)
QUEUE_NAME = 'user_queue'


def push_to_queue(user_data: User) -> None:
    serialized_data = user_data.model_dump_json()
    r.rpush(QUEUE_NAME, serialized_data)
    print(f'Added to queue: {serialized_data}')


user1 = User(id=1, name='John Doe', email='john.doe@example.com')
user2 = User(id=2, name='Jane Doe', email='jane.doe@example.com')

push_to_queue(user1)
#> Added to queue: {"id":1,"name":"John Doe","email":"john.doe@example.com"}

push_to_queue(user2)
#> Added to queue: {"id":2,"name":"Jane Doe","email":"jane.doe@example.com"}


def pop_from_queue() -> None:
    data = r.lpop(QUEUE_NAME)

    if data:
        user = User.model_validate_json(data)
        print(f'Validated user: {repr(user)}')
    else:
        print('Queue is empty')


pop_from_queue()
#> Validated user: User(id=1, name='John Doe', email='john.doe@example.com')

pop_from_queue()
#> Validated user: User(id=2, name='Jane Doe', email='jane.doe@example.com')

pop_from_queue()
#> Queue is empty
```

## RabbitMQ — sender

AMQP via `pika`. Declare the queue, publish the JSON body:

```python
import pika

from pydantic import BaseModel, EmailStr


class User(BaseModel):
    id: int
    name: str
    email: EmailStr


connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
QUEUE_NAME = 'user_queue'
channel.queue_declare(queue=QUEUE_NAME)


def push_to_queue(user_data: User) -> None:
    serialized_data = user_data.model_dump_json()
    channel.basic_publish(
        exchange='',
        routing_key=QUEUE_NAME,
        body=serialized_data,
    )
    print(f'Added to queue: {serialized_data}')


user1 = User(id=1, name='John Doe', email='john.doe@example.com')
user2 = User(id=2, name='Jane Doe', email='jane.doe@example.com')

push_to_queue(user1)
#> Added to queue: {"id":1,"name":"John Doe","email":"john.doe@example.com"}

push_to_queue(user2)
#> Added to queue: {"id":2,"name":"Jane Doe","email":"jane.doe@example.com"}

connection.close()
```

## RabbitMQ — receiver

Validate **before** acking — if the payload is malformed, an exception escapes and the message is requeued or DLQ'd instead of being silently dropped:

```python
import pika

from pydantic import BaseModel, EmailStr


class User(BaseModel):
    id: int
    name: str
    email: EmailStr


def main():
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()
    QUEUE_NAME = 'user_queue'
    channel.queue_declare(queue=QUEUE_NAME)

    def process_message(
        ch: pika.channel.Channel,
        method: pika.spec.Basic.Deliver,
        properties: pika.spec.BasicProperties,
        body: bytes,
    ):
        user = User.model_validate_json(body)
        print(f'Validated user: {repr(user)}')
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(queue=QUEUE_NAME, on_message_callback=process_message)
    channel.start_consuming()


if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        pass
```

## ARQ — async Redis job queue

ARQ jobs encode arguments as msgpack, so use `model_dump()` (Python dict) on enqueue and `model_validate()` inside the worker:

```python
import asyncio
from typing import Any

from arq import create_pool
from arq.connections import RedisSettings

from pydantic import BaseModel, EmailStr


class User(BaseModel):
    id: int
    name: str
    email: EmailStr


REDIS_SETTINGS = RedisSettings()


async def process_user(ctx: dict[str, Any], user_data: dict[str, Any]) -> None:
    user = User.model_validate(user_data)
    print(f'Processing user: {repr(user)}')


async def enqueue_jobs(redis):
    user1 = User(id=1, name='John Doe', email='john.doe@example.com')
    user2 = User(id=2, name='Jane Doe', email='jane.doe@example.com')

    await redis.enqueue_job('process_user', user1.model_dump())
    print(f'Enqueued user: {repr(user1)}')

    await redis.enqueue_job('process_user', user2.model_dump())
    print(f'Enqueued user: {repr(user2)}')


class WorkerSettings:
    functions = [process_user]
    redis_settings = REDIS_SETTINGS


async def main():
    redis = await create_pool(REDIS_SETTINGS)
    await enqueue_jobs(redis)


if __name__ == '__main__':
    asyncio.run(main())
```

⚠️ Worker functions receive `user_data` as a `dict`, not a `User` — the model is reconstructed inside the job, not at the framework boundary. Forgetting to `model_validate()` here means you are operating on raw, unvalidated input.

💡 **Takeaway:** queues are wire formats — serialize at the edge with `model_dump_json` / `model_dump`, revalidate on the consumer, and your worker code can treat every message as a typed model.
