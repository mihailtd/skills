---
name: database-redis-count-min-sketch

description: Guides the agent on Redis's Count-Min Sketch (CMS.INITBYDIM/CMS.INCRBY/CMS.QUERY/CMS.MERGE) — approximate frequency counting over a high-volume stream, in fixed memory regardless of how many distinct items appear — for "how many times has X occurred" questions, distinct from HyperLogLog's "how many distinct items" question.
---

# Redis Count-Min Sketch

You are an expert in Redis's Count-Min Sketch (CMS) probabilistic data structure. When a user needs to count *how many times* individual items occur in a high-volume stream — not how many distinct items exist (that's HyperLogLog) and not membership (that's a Bloom/Cuckoo filter) — guide them to CMS.

- **Don't confuse this with HyperLogLog.** HyperLogLog (`database-redis-probabilistic-data-structures`) answers "how many *distinct* items have I seen" — one number for the whole set. Count-Min Sketch answers "how many times has *this specific item* occurred" — a per-item frequency, queryable for any item after the fact. These are genuinely different questions with different structures; picking HyperLogLog when the real need is per-item frequency (or vice versa) doesn't produce a wrong answer to your actual question, it produces a correct answer to a different question.
- **Initialize by explicit dimensions, sized for accuracy needs, before adding data:**

  ```
  CMS.INITBYDIM grocery:25042023 50 5
  ```

  The two dimensions have a direct meaning: **width** is the number of counters per row (roughly, how many buckets items can land in), **depth** is the number of independent hash functions/rows used. Increasing depth reduces the probability that an item with a genuinely low count collides with (and gets confused for) items with a high count — i.e. more depth means better accuracy for distinguishing real signal from hash collisions, at the cost of more memory and more per-operation work to update/query every row.
  - An alternative initializer, `CMS.INITBYPROB`, lets you specify the desired error rate/probability directly instead of raw width/depth — use whichever framing matches how the requirement was actually specified (a memory budget → dimensions; an accuracy target → probability).
- **Two sketches can only be merged (`CMS.MERGE`) if they have identical dimensions** — width and depth must match exactly. If sketches might need to be combined later (e.g. per-hour sketches rolled up into a per-day sketch), pick and fix the dimensions up front across every sketch instance that might ever be merged, rather than sizing each one independently and discovering the mismatch when a merge is attempted.
- **Record occurrences with `CMS.INCRBY`** (an item can be incremented by more than 1 in one call, useful when a single event represents multiple occurrences):

  ```
  CMS.INCRBY grocery:25042023 orange 1
  CMS.INCRBY grocery:25042023 orange 5
  ```

- **Query an item's estimated frequency with `CMS.QUERY`**:

  ```
  CMS.QUERY grocery:25042023 orange   -- 6
  ```

- **The estimate is always biased upward, never downward, and the error is worse for low-frequency items.** A Count-Min Sketch can only overcount (hash collisions with other items inflate a count) — it never undercounts. This means low counts are the least trustworthy numbers this structure produces: a count near the error threshold could be entirely noise from collisions rather than real occurrences. Treat small counts with suspicion and focus interpretation on items with counts well above the noise floor — this is a structure best suited to identifying and ranking *significant* frequencies in a high-volume stream, not for precisely counting rare events.
- **Use cases**: counting occurrences in a high-cardinality, high-volume stream where exact per-item counters (one Redis key per item, or a Hash field per item) would cost too much memory — word/query frequency in a text/search stream, product sales counts, API endpoint hit counts, network flow frequency by source. If the actual need is "give me the highest-frequency items, ranked," consider `database-redis-topk` instead — Top-K is purpose-built for that, whereas CMS only answers "what's this one item's frequency" per query and doesn't natively rank the whole set for you.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def create_frequency_sketch(key: str, width: int, depth: int) -> None:
    await client.cms().initbydim(key, width, depth)

async def record_occurrence(key: str, item: str, count: int = 1) -> int:
    result = await client.cms().incrby(key, [item], [count])
    return result[0]

async def estimated_frequency(key: str, item: str) -> int:
    result = await client.cms().query(key, item)
    return result[0]

async def merge_hourly_into_daily(daily_key: str, *hourly_keys: str) -> None:
    """Requires every source sketch to share the daily sketch's exact dimensions."""
    await client.cms().merge(daily_key, len(hourly_keys), list(hourly_keys))
```
