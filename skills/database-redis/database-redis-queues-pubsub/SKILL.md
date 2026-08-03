---
name: database-redis-queues-pubsub

description: Guides the agent on Lists as FIFO queues, Sorted Sets as priority queues, and Pub/Sub for fire-and-forget broadcast — and on the shared limitation that makes all three unsuitable when message loss on a crashed/disconnected consumer is unacceptable, in which case see database-redis-streams instead.
---

# Redis Queues and Pub/Sub

You are an expert in Redis messaging patterns. When a user needs a queue or broadcast mechanism between producers and consumers, guide them to the structure that matches the ordering/priority requirement — but make sure they understand the delivery guarantee (or lack of one) before committing, since this is the single most important decision-driver among these options.

- **The three simple messaging structures share one critical limitation: none of them acknowledge delivery.** Once a message is popped (List/Sorted Set) or delivered (Pub/Sub), Redis considers it gone — if the consumer crashes after receiving it but before finishing work on it, the message is lost with no way to recover or redeliver it. This is the deciding factor for whether these are the right tool: **if losing an in-flight message on a consumer crash is unacceptable, use `database-redis-streams` instead**, which supports acknowledgment and redelivery. Don't reach for these structures for anything where message loss has a real cost (payments, order processing) — reach for them when loss is tolerable and simplicity/throughput matter more.
- **Lists — FIFO queue, preserves insertion order:**

  ```
  LPUSH queue item1 item2 item3
  RPOP queue
  ```

  Push on one end, pop from the other, for strict first-in-first-out ordering. Good for decoupling producers from consumers when processing order matters and occasional message loss on a crash is acceptable — e.g. a background job queue where a lost job just means it needs to be re-enqueued by some other mechanism, not silently forgotten forever.
- **Sorted Sets — priority queue, consumers pop the highest/lowest priority item regardless of insertion order:**

  ```
  ZADD priority_queue 1 item1 2 item2 4 item4
  ZPOPMAX priority_queue
  ```

  Use when *priority* matters more than arrival order — e.g. urgent support tickets should be processed before routine ones regardless of which was filed first. `ZPOPMIN`/`ZPOPMAX` atomically remove and return the item, so multiple consumers popping concurrently each get a distinct item, not the same one.
- **Pub/Sub — broadcast to every currently-connected subscriber, fire-and-forget:**

  ```
  SUBSCRIBE lunch_channel
  PUBLISH lunch_channel "client:34:ready"
  ```

  Every subscriber connected *at the moment of publish* receives the message; there is no queue behind it, so a subscriber that's disconnected (even briefly) simply never sees messages published during that gap — there is no history to catch up on when it reconnects. This makes Pub/Sub a good fit for live status updates, alerts, or dashboards where only the *current* state matters (a stale/missed update is superseded by the next one anyway), and a poor fit for anything where every message must eventually be processed by someone.
- **None of these three support consumer groups (distributing a stream of work across multiple independent workers, each seeing only their share) or replay (a reconnecting consumer catching up on what it missed).** Both are genuine capabilities Streams provide — see `database-redis-streams` when either is needed.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def enqueue(queue_key: str, item: str) -> None:
    await client.lpush(queue_key, item)

async def dequeue(queue_key: str) -> str | None:
    """FIFO: items pop in the order they were pushed."""
    return await client.rpop(queue_key)

async def enqueue_with_priority(queue_key: str, item: str, priority: float) -> None:
    await client.zadd(queue_key, {item: priority})

async def dequeue_highest_priority(queue_key: str) -> tuple[str, float] | None:
    result = await client.zpopmax(queue_key)
    return result[0] if result else None

async def publish_event(channel: str, message: str) -> int:
    """Returns the number of subscribers that received it — 0 means no one was listening."""
    return await client.publish(channel, message)

async def subscribe_and_handle(channel: str, handler) -> None:
    pubsub = client.pubsub()
    await pubsub.subscribe(channel)
    async for message in pubsub.listen():
        if message["type"] == "message":
            await handler(message["data"])
```
