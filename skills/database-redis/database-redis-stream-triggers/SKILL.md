---
name: database-redis-stream-triggers

description: Guides the agent on Redis Stack stream triggers (registerStreamTrigger) — a server-side consumer that runs a JavaScript callback automatically for every entry added to a matching Stream, with at-least-once delivery on shard failure — versus client-pulled consumer groups (database-redis-streams) for the same underlying data structure.
---

# Redis Stack Stream Triggers

You are an expert in Redis Stack's stream triggers. When a user needs Redis to process Stream entries automatically, in-database, as they arrive — rather than a client explicitly polling/reading the stream (see `database-redis-streams`) — guide them to `registerStreamTrigger`.

- **A stream trigger is a server-side consumer, not a client-side one.** `database-redis-streams` covers the client-pulled model (`XREADGROUP`, consumer groups, explicit `XACK`) where application code decides when to read. A stream trigger instead runs a callback *automatically*, in the database process itself, for every new entry — removing the need for a separate consumer process to poll the stream at all. Choose this when the processing logic belongs close to the data and doesn't need a general-purpose application runtime (e.g. transforming and indexing incoming events), and the client-pulled model when consumers need more control over pacing, scaling, or run outside the database as independent services.
- **Register with a consumer name, a stream (name prefix), a callback, and tuning options:**

  ```javascript
  redis.registerStreamTrigger(
      "consumer",
      "tickets",
      function(client, data) {
          redis.log(JSON.stringify(data, (key, value) =>
              typeof value === 'bigint' ? value.toString() : value
          ));
          client.call("INCR", "ntickets");
      },
      { isStreamTrimmed: false, window: 3 }
  );
  ```

  - **`consumer`** — the name identifying this registered consumer.
  - **`stream`** — the prefix matching stream key names that should trigger this callback (the same prefix-matching convention as keyspace triggers and `FT.CREATE` index prefixes).
  - **`callback`** — invoked per stream entry; can be written synchronously or asynchronously, following the same sync-vs-async execution rules as any other JavaScript function (see `database-redis-javascript-functions`) on whichever shard actually owns and stores the stream.
  - **`window`** — how many entries can be processed concurrently/in-flight at once — a throughput/concurrency tuning knob, not a correctness one.
  - **`isStreamTrimmed`** — whether processed entries should be trimmed from the stream afterward. Set `true` when the trigger is the *only* consumer of this stream's data and there's no reason to retain entries once processed; leave `false` when other consumers (client-pulled consumer groups, other triggers) also need to read the same entries, or historical replay is a requirement.
- **Test by adding entries with `XADD` as usual** — the trigger fires without any explicit read call:

  ```
  XADD tickets * movie "The Godfather" paid "35"
  ```

  The callback receives the entry's id, stream name, and field/value pairs as `data` — inspect via `redis.log(JSON.stringify(data))` while developing to see the actual shape delivered.

## Delivery Guarantee

- **While the shard storing the stream is healthy, each entry's callback runs exactly once.** If that shard crashes and recovers, the guarantee weakens to *at least once* — a callback may re-run for an entry it had already processed before the crash. **Design callback logic to be idempotent** (safe to run twice with the same entry) if the stream trigger might ever run against a deployment where shard failure is a real possibility — don't assume "exactly once" holds unconditionally just because it's the common case.

## Use Cases

- **Real-time transformation and indexing**: consume raw events from a stream and write them into indexed JSON/Hash documents (see `database-redis-json-documents`/`database-redis-search-indexing`) as they arrive, turning Redis into an ingest-transform-index pipeline without a separate stream-processing service.
- Anywhere the processing logic is simple/bounded enough to run in-database and the value of removing an external consumer process outweighs the flexibility of a general-purpose one — for heavier or more complex stream processing (joins across streams, long-running stateful aggregation), a dedicated external consumer using `database-redis-streams`'s client-pulled model is likely still the better fit.

## Code Example

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

STREAM_TRIGGER_LIBRARY = """
#!js api_version=1.0 name=ticketprocessor
redis.registerStreamTrigger(
    "consumer",
    "tickets",
    function(client, data) {
        client.call("INCR", "ntickets");
    },
    { isStreamTrimmed: false, window: 3 }
);
"""

async def load_stream_trigger() -> None:
    await client.execute_command("TFUNCTION", "LOAD", "REPLACE", STREAM_TRIGGER_LIBRARY)

async def publish_ticket(movie: str, paid: str) -> str:
    """The trigger fires automatically — no explicit read/consume call needed."""
    return await client.xadd("tickets", {"movie": movie, "paid": paid})
```
