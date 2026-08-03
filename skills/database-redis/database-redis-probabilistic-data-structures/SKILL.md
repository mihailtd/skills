---
name: database-redis-probabilistic-data-structures

description: Guides the agent on trading exact accuracy for memory/speed at scale using Redis's probabilistic data structures — HyperLogLog (PFADD/PFMERGE) for cardinality estimation, Bloom filters (BF.ADD/BF.RESERVE error-rate tuning) for membership/deduplication testing, bitfields and bitmaps for compact counters and flags — and routing to the right sketch (Cuckoo filter, Count-Min Sketch, Top-K, t-digest) for other question shapes.
---

# Redis Probabilistic Data Structures

You are an expert in Redis's probabilistic data structures. When a user needs to answer a question over a very large or unbounded dataset — unique counts, membership, top-k, frequency — help them recognize when an approximate, constant-memory answer is the right trade-off against an exact structure that grows linearly with data volume.

- **The trade-off these structures make is explicit: give up exact accuracy, gain near-constant memory and time regardless of how large the underlying dataset grows.** This is worth it whenever the exact answer isn't actually required — "roughly how many unique visitors" is usually just as actionable as "exactly how many," but costs orders of magnitude less to compute at scale.
- **Recognize the shape of a question that calls for a probabilistic structure**, rather than reaching straight for an exact deterministic one and hitting a memory wall later — route each shape to the specific sketch built for it, not a generic "probabilistic" catch-all:
  - "How many distinct values have I seen?" → cardinality estimation — **HyperLogLog** (this skill, below).
  - "Has this value already appeared?" (e.g. has this user already seen this ad) → membership testing — **Bloom filter** (this skill, below) if items are only ever added/checked, never removed; **Cuckoo filter** (`database-redis-cuckoo-filter`) if items need to be deletable or counted.
  - "How many times has this specific value appeared?" → frequency estimation over a stream — **Count-Min Sketch** (`database-redis-count-min-sketch`).
  - "What are the top-N most frequent items?" → ranked frequency tracking — **Top-K** (`database-redis-topk`).
  - "What fraction of values falls below/above X?" / "What's the Nth percentile?" → quantile estimation — **t-digest** (`database-redis-tdigest`).
  - Picking the wrong sketch for the question is a common mistake — e.g. reaching for HyperLogLog (cardinality) when the real question is frequency (Count-Min Sketch), or building a Bloom filter when items need to be removable (Cuckoo filter's actual job). Match the question shape above before writing any code.
- **HyperLogLog (`PFADD`/`PFCOUNT`) estimates set cardinality in a small, effectively fixed amount of memory**, regardless of how many elements have been added — this is the key property that distinguishes it from an exact `SADD`/`SCARD` approach, whose memory grows linearly with the number of unique elements stored:

  ```
  PFADD pages "https://redis.com/" "https://redis.io/docs/stack/bloom/"
  PFCOUNT pages
  ```

  The accuracy trade-off is a small, well-characterized standard error (on the order of ~0.8%) — acceptable for analytics/monitoring use cases, not acceptable where the exact count has a hard downstream consequence (e.g. billing).
- **Combine multiple HyperLogLogs into a union cardinality with `PFMERGE`**, instead of re-adding every element from every source into one structure — e.g. "unique visitors across January and February" from two monthly HyperLogLogs:

  ```
  PFMERGE pages:2023 pages:012023 pages:022023
  PFCOUNT pages:2023
  ```

  This is genuinely a set union, not a sum of the individual counts — `PFCOUNT` on the merged result correctly accounts for visitors present in both source periods, rather than double-counting them the way adding the two `PFCOUNT` results naively would.
- **Don't reach for an exact Set just to avoid the "approximate" label when the actual requirement tolerates it.** An exact `SADD` of hashed values gives a precise cardinality via `SCARD`, but its memory cost scales with the number of unique members stored — for large or unbounded cardinalities, this can be many times the memory HyperLogLog uses for the same estimate. Measure both before assuming exactness is "free."
- **For compact exact counters — not estimation, just efficient storage of many small counters — use `BITFIELD`** to pack multiple fixed-width integer counters into a single string value at different bit offsets, instead of one Redis key per counter. This is a different technique from HyperLogLog (it's still exact), useful when the real cost you're optimizing is key/memory overhead from having many separate counter keys, not accuracy.
- **For a single flag per unit of time (not a counter, just "did this happen"), use `SETBIT`/`BITCOUNT` instead of `BITFIELD`.** This is a distinct pattern from packed counters: one bit per day/hour/slot records presence/absence, and `BITCOUNT` over a range answers "how many of these slots were flagged" in one call — e.g. flagging which days of a month a user authenticated:

  ```
  SETBIT user:032023 0 1
  SETBIT user:032023 5 1
  SETBIT user:032023 10 1
  BITCOUNT user:032023 0 31
  ```

  This is exact (not probabilistic) and extremely compact — a full month of daily flags fits in well under 100 bytes regardless of how many days are set — so reach for it whenever the question is "on which of N discrete slots did X happen" and a full Set of timestamps or a counter-per-day key would be overkill.
- **Bloom filters (`BF.ADD`/`BF.EXISTS`, or `BF.MADD`/`BF.MEXISTS` for multiple items in one call) are the probabilistic answer to "have I seen this before" (deduplication/membership testing)**, storing only a hashed representation of each item instead of the item itself:

  ```
  BF.ADD visited www.redis.com
  BF.EXISTS visited www.learn.redis.com
  ```

  The asymmetry to internalize: **false positives are possible** ("exists" can wrongly say yes), **but false negatives are not** — if `BF.EXISTS` says an item was never added, that's certain. This makes Bloom filters safe for "skip expensive work if we're sure we've already done it" (a false positive just means occasionally not skipping when you could have) but unsafe for "reject this because it's a duplicate" if a false positive would incorrectly block something that's actually new — know which direction the error tolerance needs to point before choosing this over an exact `SADD`/`SISMEMBER`.
  - `BF.MADD`/`BF.MEXISTS` let you check/add several items against the same filter atomically in one round trip — useful for compound membership checks (e.g. "has this exact combination of location and hour been seen for this user before") without multiple separate calls.
  - **`BF.CARD` reports the number of items added to the filter** — this is exact bookkeeping the filter maintains regardless of its approximate membership testing, useful for monitoring how full a filter is getting, independent of whether any particular membership check might be a false positive.
  - **Tune the false-positive rate explicitly with `BF.RESERVE`** instead of accepting whatever the default gives you, when the use case has a specific accuracy requirement: `BF.RESERVE key error_rate capacity` pre-sizes the filter for an expected item count at a chosen error rate. **Lowering the error rate costs more memory and more computation per operation** (more/larger hash functions) — there's no free lunch, so pick the error rate the use case actually needs (e.g. a spam-filter false-positive costs a mildly annoying false flag; a "already processed this payment" check might warrant a much lower error rate) rather than defaulting to the tightest possible setting everywhere.
  - **Bloom filters cannot delete items** — once added, an item can never be removed from a Bloom filter (only recreated from scratch). If deletion is a real requirement, that alone is reason enough to reach for a **Cuckoo filter** (`database-redis-cuckoo-filter`) instead, even though Bloom filters are otherwise the simpler, more common choice for plain membership testing.
- **Common Bloom-filter use cases beyond deduplication**: spam/malicious-URL or IP blocklists, fraud-pattern/stolen-card checks (shareable between organizations without exposing the underlying sensitive values, since only a hashed representation is stored), suspicious-activity/geolocation verification (see `database-redis-fraud-detection`), "has this user seen this ad" ad-frequency capping, and spellchecking (a word not in the filter is definitely misspelled).

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def track_unique_page_view(user_pages_key: str, url: str) -> None:
    """Approximate unique-visitor tracking with near-constant memory."""
    await client.pfadd(user_pages_key, url)

async def estimated_unique_pages(user_pages_key: str) -> int:
    return await client.pfcount(user_pages_key)

async def compact_counter_increment(counter_key: str, amount: int = 1) -> int:
    """Pack a small counter into a bit-offset within one string value."""
    bf = client.bitfield(counter_key)
    bf.incrby("u16", 0, amount)
    result = await bf.execute()
    return result[0]

async def flag_login_day(user_month_key: str, day_of_month: int) -> None:
    """One bit per day: exact, and extremely compact for presence/absence tracking."""
    await client.setbit(user_month_key, day_of_month, 1)

async def logins_this_month(user_month_key: str) -> int:
    return await client.bitcount(user_month_key, 0, 31)

async def mark_url_visited(dedup_key: str, url: str) -> None:
    """Bloom filter: store only a hashed representation, not the full URL."""
    await client.bf().add(dedup_key, url)

async def was_url_visited(dedup_key: str, url: str) -> bool:
    """No false negatives — a False result is certain; a True result may be a false positive."""
    return bool(await client.bf().exists(dedup_key, url))

async def merge_monthly_uniques(dest_key: str, *source_keys: str) -> int:
    """PFMERGE: union cardinality across periods, no double-counting."""
    await client.pfmerge(dest_key, *source_keys)
    return await client.pfcount(dest_key)

async def create_tuned_filter(key: str, error_rate: float, expected_items: int) -> None:
    """Pre-size and tune accuracy explicitly instead of accepting the default."""
    await client.bf().create(key, error_rate, expected_items)

async def filter_item_count(key: str) -> int:
    """Exact count of items added — independent of any approximate membership check."""
    return await client.bf().card(key)
```
