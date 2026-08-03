---
name: database-redis-rate-limiting

description: Guides the agent on implementing rate limiters in Redis — a fixed-window counter (INCR/EXPIRE) for the simple case, and a sliding-window Sorted Set for accurate or weighted limiting — done atomically to avoid race conditions between the check and the increment.
---

# Redis Rate Limiting

You are an expert in implementing rate limiters with Redis. When a user needs to throttle requests, logins, or any other action per identity (user, API key, IP) within a time window, guide them to the technique that matches their accuracy needs, and make sure the check-and-increment is atomic.

- **A rate limiter is a counter with an expiration, scoped per identity and time bucket.** The key name should encode both: `{identity}:{time-bucket}` (e.g. `api-key-123:14:32` for a per-minute limiter). This means old buckets clean themselves up via TTL — you never need a separate job to prune expired counters.
- **Fixed-window counter — simplest, cheapest, and usually good enough:**

  ```
  INCR api-key-123:14:32
  EXPIRE api-key-123:14:32 59
  ```

  Increment first, check the returned value against the threshold, and set an expiration only on the first increment in a bucket (or unconditionally with a value just over the bucket duration — re-setting the same TTL repeatedly is harmless). Bucketing by a fixed unit (the current minute) means the counter for a past bucket is guaranteed expired long before that bucket number recurs — you don't need to track "which minute" beyond the current value.
  - **Known limitation, worth stating explicitly to whoever's relying on this:** a fixed window allows up to 2x the intended rate at the boundary — e.g. a burst at the end of minute N and another at the start of minute N+1 can both hit the limit independently, giving up to double the allowed rate across that 1-2 second boundary. Acceptable for most abuse-prevention/DoS-mitigation use cases; not acceptable when the limit is a hard contractual guarantee.
- **Sliding-window Sorted Set — accurate, and supports per-request weighting:**

  ```
  ZREMRANGEBYSCORE api-key-123 0 <now_ms - window_ms>
  ZADD api-key-123 <now_ms> <now_ms>:<weight>
  ZRANGE api-key-123 0 -1
  EXPIRE api-key-123 <window_seconds>
  ```

  Each request adds a member scored by its own timestamp; `ZREMRANGEBYSCORE` trims anything older than the window on every call, so the Sorted Set always contains exactly the requests still inside the sliding window — no boundary-doubling artifact like the fixed-window approach. Read back the members (`ZRANGE`, or `ZCOUNT` for an unweighted count) and sum weights against the threshold in the caller. Give every member a unique value (timestamp + a suffix, e.g. `:weight` or a request ID) — two requests landing on the exact same millisecond would otherwise collide as the same Sorted Set member and only count once.
  - Use the weight suffix to implement a **weighted limiter**, where some operations cost more of the budget than others (e.g. a bulk export costing 5 "requests" against a per-day quota) — omit it (or fix it at 1) for a plain per-request count.
- **Every rate limiter must check-and-increment atomically, or concurrent requests can both pass a check that should have rejected the second one.** Wrap the read-modify-write sequence in `MULTI`/`EXEC` (or a pipeline in transactional mode) — never issue the `INCR`/`ZADD` and the threshold check as two separate, un-transacted round trips where a race is possible between them.
- **Run the rate limiter as close to the entry point as the architecture allows** — at an API gateway or load balancer, before the request reaches application servers — so rejected requests never consume backend capacity at all. For a multi-node gateway cluster, the limiter must be externalized (not per-node in-memory state) precisely because it needs one shared view of the count across every node handling that identity's traffic; this is the natural fit for Redis as a fast, centralized counter store.
- **Choose fixed-window by default; move to sliding-window only when the boundary-doubling behavior is actually unacceptable**, or when weighted limiting is a real requirement — the sliding-window approach costs more (a Sorted Set entry per request, trimmed on every call, versus one integer) for the accuracy it buys.

## Code Examples

```python
from redis import asyncio as aioredis
import time

client = aioredis.from_url("redis://localhost")

async def fixed_window_allow(identity: str, limit: int, window_seconds: int = 60) -> bool:
    """Simple per-minute counter; allows brief bursts at window boundaries."""
    bucket = int(time.time() // window_seconds)
    key = f"{identity}:{bucket}"
    async with client.pipeline(transaction=True) as pipe:
        pipe.incr(key)
        pipe.expire(key, window_seconds - 1)
        count, _ = await pipe.execute()
    return count <= limit

async def sliding_window_allow(identity: str, limit: int, window_ms: int = 86_400_000, weight: int = 1) -> bool:
    """Accurate sliding window with per-request weighting."""
    now_ms = int(time.time() * 1000)
    member = f"{now_ms}:{weight}"
    async with client.pipeline(transaction=True) as pipe:
        pipe.zremrangebyscore(identity, 0, now_ms - window_ms)
        pipe.zadd(identity, {member: now_ms})
        pipe.zrange(identity, 0, -1)
        pipe.expire(identity, window_ms // 1000)
        *_, members, _ = await pipe.execute()
    total_weight = sum(int(m.decode().split(":")[1]) for m in members)
    return total_weight <= limit
```
