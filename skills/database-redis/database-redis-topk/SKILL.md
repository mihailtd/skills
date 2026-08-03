---
name: database-redis-topk

description: Guides the agent on Redis's Top-K structure (TOPK.RESERVE/TOPK.INCRBY/TOPK.LIST) — approximate, fixed-memory tracking of the K highest-frequency items in a stream via the HeavyKeeper algorithm — for leaderboards, trending content, and anomaly/DDoS detection at scale where an exact Sorted Set would cost too much memory.
---

# Redis Top-K

You are an expert in Redis's Top-K probabilistic data structure. When a user needs "the K most frequent items" from a high-volume or high-cardinality stream — not every item's exact count, just the ranked leaders — guide them to Top-K instead of an exact Sorted Set once the item cardinality makes an exact structure too expensive.

- **Top-K answers a different question than a Sorted Set does, even though both can produce a "leaderboard."** A Sorted Set (`database-redis-leaderboards`) is exact and lets you query any member's exact score/rank, at a memory cost proportional to the *total number of distinct members ever added*. Top-K only ever tracks the current top K items and their approximate counts — it forgets everything else — at a fixed memory cost regardless of how many distinct items have streamed through. Reach for Top-K specifically when the item cardinality is large/unbounded (e.g. every URL ever requested, every IP address seen) and only the current leaders actually matter, not the full ranked list.
- **Initialize with the number of top items to track, plus tunable collision-resistance parameters:**

  ```
  TOPK.RESERVE players 10
  TOPK.RESERVE players 10 100 5 0.9   -- topk width depth decay, explicit
  ```

  - **`topk`** — how many top items to retain; this bounds memory usage regardless of total stream volume.
  - **`width`**/**`depth`** — the same two-dimensional-array collision-resistance concept as `database-redis-count-min-sketch`: width is the counter-array size, depth is the number of hash functions. A reasonable starting point is `width = topk * log(topk)` and `depth = log(topk)` (minimum 5) — tune from there based on measured accuracy against your actual data, not by assuming the defaults fit every workload.
  - **`decay`** — controls how the HeavyKeeper algorithm resolves collisions (see below); `0.9` is a reasonable starting default.
- **Record occurrences with `TOPK.INCRBY`** (an item can be incremented by more than one occurrence per call, same as Count-Min Sketch):

  ```
  TOPK.INCRBY players john 5
  TOPK.INCRBY players tod 10
  ```

- **Retrieve the current ranked leaders with `TOPK.LIST WITHCOUNT`**:

  ```
  TOPK.LIST players WITHCOUNT
  ```

  Returns the current top-K items in ranked order with their approximate counts — this is the structure's whole point: no separate ranking/sort step needed, the structure maintains rank incrementally as items are incremented.
- **How collisions are resolved (the HeavyKeeper algorithm) is worth understanding because it explains the structure's actual behavior under load**: when two items collide in the same bucket, the *existing* occupant's counter is probabilistically decayed rather than unconditionally evicted — and the probability of that decay happening is itself a function of the occupant's current count. High-count items ("elephant flows") are increasingly resistant to eviction as their count grows; low-count items ("mouse flows") are easily displaced. This is what makes the structure good at *keeping* genuinely high-frequency items stable in the top-K even under heavy collision pressure from a long tail of low-frequency noise — it's not a naive LRU-style eviction.
- **Use cases**: DDoS/network-anomaly detection (which source IPs/addresses have the highest request-rate surge — "give me the current attackers," not "give me every IP's exact count"), memory-efficient leaderboards at very large user counts (trade exact full-ranking for a bounded-memory "who's currently winning"), and trending-content detection (trending hashtags, trending pages/videos) where only the current leaders matter and yesterday's leaders can be forgotten.
- **If the requirement needs an item's exact count, or needs to query ranks/counts for items *outside* the current top K, Top-K is the wrong structure** — it has genuinely forgotten anything not currently in its bounded top-K set. Use `database-redis-count-min-sketch` for "give me this specific item's approximate frequency" (any item, not just current leaders), or an exact Sorted Set (`database-redis-leaderboards`) when every member's exact score truly needs to be queryable.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def create_topk(key: str, k: int) -> None:
    await client.topk().reserve(key, k)

async def record_event(key: str, item: str, count: int = 1) -> None:
    await client.topk().incrby(key, [item], [count])

async def current_leaders(key: str) -> list[tuple[str, int]]:
    """Ranked leaders with counts — no separate sort/rank step needed."""
    result = await client.topk().list(key, withcount=True)
    return list(zip(result[0::2], result[1::2]))
```
