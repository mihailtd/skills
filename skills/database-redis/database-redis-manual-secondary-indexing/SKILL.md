---
name: database-redis-manual-secondary-indexing

description: Guides the agent away from full-keyspace SCAN filtering when Redis Stack's RediSearch isn't available, toward pipelining, server-side Lua functions, or hand-rolled Set/Sorted-Set indexes kept consistent with MULTI/EXEC — and toward recognizing when to stop hand-rolling and adopt RediSearch instead.
---

# Redis Manual Secondary Indexing (No RediSearch)

You are an expert in Redis performance and data modeling. When a user needs to filter or search Redis data on a non-key attribute and does not have (or hasn't yet reached for) Redis Stack's search module, use the following guidance to avoid full-keyspace scans and to choose the least-bad workaround for their constraints.

- **Redis has no built-in secondary index on core data structures.** A Hash's fields are only ever looked up by the Hash's own key — there is no way to ask "find every Hash where field X equals Y" without either scanning the keyspace or maintaining your own index structure.
- **Never use `KEYS` to enumerate matching keys in anything but a one-off admin script.** `KEYS` is O(N) and blocks the single-threaded server for its entire duration, freezing every other client. Always use `SCAN` (with `MATCH` and a reasonable `COUNT`) instead — it's cursor-based and non-blocking, returning results incrementally without holding up the server.
- **A raw `SCAN` + per-key filter is still fundamentally a full-keyspace linear scan**, whether the filtering logic runs in the client or the server — it just changes *where* the O(N) cost is paid, not whether it's paid. Treat this as a last resort, not a pattern to build production features on, especially as the keyspace grows past a few thousand keys of the relevant type.
- **When RediSearch is unavailable and you must reduce the cost of this scan, pick based on what's actually the bottleneck:**

  1. **Pipelining** — batch multiple commands into one round trip. Doesn't reduce the number of operations the server performs, but eliminates per-command network latency, which is often the dominant cost when many small commands run sequentially. Downsides: response buffering adds server memory pressure proportional to the batch size, and the client has to handle a batch of results (including partial failures) instead of one result at a time.
  2. **Server-side Lua functions** (`FUNCTION LOAD` + `FCALL`) — move the scan-and-filter loop to run on the server, next to the data, eliminating per-key round trips entirely. The trade-off is that Lua execution is single-threaded and blocking: a long-running function makes the *entire* server unresponsive to other clients for its duration, not just the calling client. Only acceptable for scans bounded to a size you've actually measured as safe, not "however big the dataset happens to grow to."
  3. **A hand-rolled secondary index** using a Set (unique membership, e.g. "all Spanish cities") or Sorted Set (membership + a ranked/range-queryable score, e.g. "Spanish cities by population") — genuinely fast reads (`SISMEMBER`, `SMEMBERS`, `ZRANGEBYSCORE` are all low-complexity), at the cost of you now owning index maintenance.

- **A hand-rolled index must be kept in sync with the data it indexes, atomically, or it silently drifts.** Any operation that touches both the entity and its index (e.g. deleting a city Hash and removing it from the `city:esp` Set) must be wrapped so both changes succeed or neither does — use `MULTI`/`EXEC` (or a Lua function, for conditional logic that a plain transaction can't express) rather than two separate round trips where a crash or race between them leaves the index pointing at data that's gone, or missing data that exists.
- **A composite/multi-field manual index (e.g. "Spanish cities with population > 2M") gets disproportionately harder to maintain** as more fields need to be indexed together — each additional filterable attribute is another structure to keep synchronized on every write. Treat "we need to filter on a second or third field" as the signal to stop hand-rolling and adopt RediSearch (see `database-redis-search-indexing`), which maintains composite indexes for you, synchronously, on every write, with no application-side bookkeeping.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def scan_and_filter_by_country(country_code: str) -> list[str]:
    """Last resort: linear scan, only for small keyspaces or one-off scripts."""
    matches = []
    async for key in client.scan_iter(match="city:*", count=100):
        code = await client.hget(key, "CountryCode")
        if code == country_code:
            matches.append(await client.hget(key, "Name"))
    return matches

async def add_city_with_index(city_id: int, name: str, country_code: str, population: int) -> None:
    """Keep a hand-rolled index consistent with the data using a transaction."""
    async with client.pipeline(transaction=True) as pipe:
        pipe.hset(f"city:{city_id}", mapping={
            "Name": name, "CountryCode": country_code, "Population": population,
        })
        if country_code == "ESP":
            pipe.zadd("city:esp", {name: population})
        await pipe.execute()

async def remove_city_with_index(city_id: int, name: str) -> None:
    """Removal must update the entity and the index atomically too, or the index goes stale."""
    async with client.pipeline(transaction=True) as pipe:
        pipe.delete(f"city:{city_id}")
        pipe.zrem("city:esp", name)
        await pipe.execute()

async def top_spanish_cities(min_population: int) -> list[str]:
    """Reading the hand-rolled index is fast — the cost was all in maintaining it."""
    return await client.zrangebyscore("city:esp", min_population, "+inf")
```
