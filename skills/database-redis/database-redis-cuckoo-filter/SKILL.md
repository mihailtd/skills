---
name: database-redis-cuckoo-filter

description: Guides the agent on Redis Cuckoo filters (CF.ADD/CF.EXISTS/CF.COUNT/CF.DEL) — a fingerprint-bucket-based alternative to Bloom filters that supports deletion and repeat-item counting, for approximate membership testing where items must be removable.
---

# Redis Cuckoo Filters

You are an expert in Redis's Cuckoo filter probabilistic data structure. When a user needs approximate membership testing ("has this been seen before") *and* the ability to remove items later, guide them to a Cuckoo filter instead of a Bloom filter (see `database-redis-probabilistic-data-structures`) — Bloom filters cannot delete items at all.

- **The deciding factor versus a Bloom filter is deletability.** Both answer the same kind of question (probable membership, with false positives possible but no false negatives) at similar space cost, but a Bloom filter's underlying representation makes removing a single item's influence on the filter impossible without rebuilding it from scratch. A Cuckoo filter stores a fingerprint of each item in a bucket, which can be explicitly removed — reach for Cuckoo whenever the set being modeled is expected to shrink as well as grow (e.g. a temporary block-list, an active-session set, anything with a real "this is no longer true" case), not just accumulate.
- **Cuckoo filters can also count repeated insertions of the same item**, which a Bloom filter has no concept of (a Bloom filter only ever answers yes/no membership, never "how many times"). This makes a Cuckoo filter useful for lightweight approximate multiplicity tracking, not just presence — though if frequency counting is the *primary* need rather than a side benefit, `database-redis-count-min-sketch` is the structure actually designed for that.
- **Add and check items with the same shape as a Bloom filter**, so migrating between them (if deletability turns out to matter later) is a low-friction swap in most cases:

  ```
  CF.ADD service:username mortensi
  CF.EXISTS service:username mortensi
  ```

- **Track and query repeat counts with `CF.COUNT`** — adding the same item again increments its count rather than being a no-op:

  ```
  CF.ADD service:username mortensi
  CF.COUNT service:username mortensi   -- 2, if added twice
  ```

- **Remove an item with `CF.DEL`**, decrementing its count (or removing it entirely once the count reaches zero) — this is the capability Bloom filters simply don't have:

  ```
  CF.DEL service:username mortensi
  CF.COUNT service:username mortensi   -- back down by one
  ```

- **How it avoids collisions internally**: instead of only setting bits in a shared bit array (a Bloom filter's approach), a Cuckoo filter stores each item's fingerprint in one of several candidate buckets. When a new item's candidate bucket is already occupied, the *existing* fingerprint gets relocated to one of its own alternate candidate buckets (the "cuckoo" behavior the structure is named for) — kicking out and re-homing the occupant rather than simply accepting the collision. This generally gives a lower false-positive rate than a Bloom filter at comparable space usage, at the cost of a marginally more complex insert path (the relocation search).
- **Use cases mirror Bloom filters' (spam/fraud blocklists, suspicious-activity checks, ad-frequency capping, spellchecking — see `database-redis-probabilistic-data-structures`), specifically wherever any of those needs items removed or re-counted over time** — e.g. a "known bad IP" blocklist that needs entries to expire/be unblocked, not just grow forever.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def register_username(filter_key: str, username: str) -> None:
    await client.cf().add(filter_key, username)

async def is_username_taken(filter_key: str, username: str) -> bool:
    """No false negatives — False is certain; True may (rarely) be a false positive."""
    return bool(await client.cf().exists(filter_key, username))

async def release_username(filter_key: str, username: str) -> None:
    """The capability a Bloom filter can't offer: remove an item's membership."""
    await client.cf().delete(filter_key, username)

async def username_occurrence_count(filter_key: str, username: str) -> int:
    return await client.cf().count(filter_key, username)
```
