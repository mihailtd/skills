---
name: database-redis-streams

description: Guides the agent on Redis Streams for durable, replayable, acknowledged inter-service messaging — XADD/XREADGROUP/XACK/XPENDING and consumer groups — as the upgrade path from Lists/Pub-Sub once message loss on a crashed consumer is unacceptable.
---

# Redis Streams

You are an expert in Redis Streams. When a user needs guaranteed-delivery, replayable, or horizontally-scaled message processing between services — and plain Lists, Sorted Sets, or Pub/Sub (see `database-redis-queues-pubsub`) don't provide strong enough delivery guarantees — guide them to Streams. This skill covers the client-pulled consumer model; when processing should instead run automatically, in-database, as entries arrive — with no separate consumer process reading the stream — see `database-redis-stream-triggers`.

- **A Stream is an append-only log, comparable in role to a Kafka topic partition**: producers append entries with `XADD`, entries persist (surviving a server restart, unlike Pub/Sub), and consumers read them independently of when they were produced — including replaying from an arbitrary point in the past, not just "what's new since I connected."
- **Append entries with `XADD`, letting Redis assign the entry ID** (`*`) unless you have a specific ordering reason not to:

  ```
  XADD lunch_channel * client:34 pasta
  ```

  The returned ID (`1676982250423-0`) is a timestamp-sequence pair that's used later to acknowledge or reference that specific entry.
- **Create a consumer group to get guaranteed at-least-once delivery per group** — this is the mechanism that makes Streams fundamentally different from Pub/Sub's fire-and-forget broadcast. Precisely: each new entry goes to exactly *one* consumer *per delivery attempt* — but if that consumer crashes before acknowledging (see the `XACK` bullet below), the same entry is redelivered to another consumer, so the guarantee across the entry's full lifecycle is at-least-once, not exactly-once. Getting an actual exactly-once *effect* out of that requires the consumer's own processing to be idempotent — see **architecture-delivery-semantics** (package `architecture`) and **database-postgresql-idempotency-keys** for the deduplication pattern that turns this at-least-once delivery into an effectively-exactly-once outcome:

  ```
  XGROUP CREATE lunch_channel waiters $ MKSTREAM
  ```

  `MKSTREAM` creates the stream if it doesn't exist yet; `$` means the group starts consuming from entries added *after* this point (use `0` to start from the very beginning of the stream's history instead).
- **Consumers in the same group read with `XREADGROUP`, using `>` to mean "give me entries never yet delivered to any consumer in this group":**

  ```
  XREADGROUP GROUP waiters Alice COUNT 1 STREAMS lunch_channel >
  ```

  Redis guarantees each entry goes to exactly one consumer within the group — two consumers calling this concurrently never receive the same entry. This is how Streams distribute work across multiple parallel workers safely, something none of the structures in `database-redis-queues-pubsub` can do.
- **A read is not the same as done — the consumer must `XACK` once processing actually completes**, or Redis considers the entry still "pending" (delivered but unconfirmed) indefinitely:

  ```
  XACK lunch_channel waiters 1676982250423-0
  ```

  This is the core guarantee Streams add over Lists/Pub-Sub: if a consumer crashes after `XREADGROUP` but before `XACK`, the entry isn't lost — it's still visible as pending and can be claimed and reprocessed (e.g. via `XCLAIM`/`XAUTOCLAIM`, or by the same consumer reconnecting and checking its own pending list).
- **Check for unacknowledged work with `XPENDING`**, per-group or filtered to a specific consumer, to answer "what was delivered but never confirmed done":

  ```
  XPENDING lunch_channel waiters
  ```

  This gives visibility that Pub/Sub and Lists simply don't have — an operator or monitoring job can detect stuck/crashed consumers by watching for entries that stay pending too long, and take corrective action (reassign, alert, retry).
- **Multiple independent consumer groups can each read the entire stream, while consumers within one group split the work.** A common architecture is one consumer group per microservice that needs to react to the same event stream — every microservice sees every event (group-level), but within a microservice, multiple worker instances split the processing load (consumer-level), matching horizontal scaling of the service itself.
- **Choose Streams over Lists/Sorted-Sets/Pub-Sub specifically when at least one of these is a real requirement**: guaranteed processing (no silent loss on consumer crash), replay from history, or independent parallel consumer groups reading the same event feed. If none of those apply and the workload is simple/tolerant of occasional loss, the simpler structures in `database-redis-queues-pubsub` are less operational overhead for the same throughput.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def ensure_consumer_group(stream_key: str, group: str) -> None:
    try:
        await client.xgroup_create(stream_key, group, id="$", mkstream=True)
    except Exception:
        pass  # group already exists

async def publish_event(stream_key: str, fields: dict) -> str:
    return await client.xadd(stream_key, fields)

async def consume_one(stream_key: str, group: str, consumer: str) -> tuple[str, dict] | None:
    """Only ever receives entries never delivered to this group before."""
    result = await client.xreadgroup(group, consumer, {stream_key: ">"}, count=1)
    if not result:
        return None
    _, entries = result[0]
    entry_id, fields = entries[0]
    return entry_id, fields

async def acknowledge(stream_key: str, group: str, entry_id: str) -> None:
    await client.xack(stream_key, group, entry_id)

async def pending_summary(stream_key: str, group: str) -> dict:
    """Visibility into delivered-but-unacknowledged entries — a crashed-consumer signal."""
    return await client.xpending(stream_key, group)
```
