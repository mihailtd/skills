---
name: database-redis-caching-strategy

description: Instructs the agent on advanced Redis caching strategies, including managing TTLs to prevent memory bloat, applying jitter to prevent cache stampedes, and using Lua scripting for atomic server-side caching.
---

# Redis Caching Strategy Guidelines

You are an expert backend developer specializing in high-performance distributed caching using Redis. When asked to implement caching mechanisms, manage key expirations, or optimize Redis interactions, you must adhere to the following rules:

- In 2026, Redis remains the right choice for extreme throughput and specialized ephemeral workloads, but it is no longer the default answer for every performance problem. Modern PostgreSQL features such as native AIO and UUIDv7 have reduced the need for sidecar caches in many applications.
- Use Redis deliberately for workloads where latency, throughput, and cache semantics are the primary bottlenecks, not simply because a cache is expected.

## 1. Manage Volatile Data with TTLs
Never store temporary or volatile data (such as user sessions or API responses) without an expiration time, as this can lead to memory bloat and out-of-memory errors [1, 2].
*   When using the Redis client, utilize commands like `SETEX` or pass the `ex` parameter to the `SET` command to automatically expire the data [2]. 

## 2. Prevent Cache Stampedes using Jitter
Avoid setting the exact same TTL for multiple keys that are created or updated simultaneously (e.g., expiring a daily news feed precisely at midnight) [3]. 
*   Simultaneous expiration causes a "cache stampede," where the application is flooded with cache misses, forcing it to violently query the primary database and potentially crash the system [3].
*   **The Jitter Solution:** To prevent stampedes, add a small random "jitter" (a slight variation in seconds) to the TTL of different keys so they expire gradually over a window of time [3].

## 3. Atomic Server-Side Caching (Lua Scripting)
When implementing the cache-aside pattern for complex or expensive computations, avoid race conditions where multiple clients might try to recompute and cache the same missing data simultaneously [4, 5].
*   Push the caching logic directly into Redis using server-side Lua Scripts [4].
*   Use the `EVAL` command to execute a script that checks if a key exists, returns it if it does, or computes/sets the new value and TTL if it doesn't [5, 6].
*   Because Redis executes Lua scripts atomically in a single thread, this eliminates race conditions and reduces network round-trips [4, 7].

---

## Code Examples

Below are best-practice examples demonstrating how to safely implement TTL jitter and atomic Lua caching using the async Python `redis` client.

**1. Preventing Cache Stampedes with TTL Jitter**
```python
import random
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def cache_daily_feed(user_id: str, feed_data: str) -> None:
    """Caches the user feed with a base TTL of 24 hours, plus up to 5 minutes of jitter."""
    base_ttl = 86400  # 24 hours in seconds
    
    # Add a random jitter (0 to 300 seconds) so not all feeds expire at the exact same second
    jitter = random.randint(0, 300)
    
    # Use the 'ex' parameter to set the value and the expiration atomically
    await client.set(f"feed:user:{user_id}", feed_data, ex=base_ttl + jitter)
```

**2. Atomic Cache-Aside Pattern with Lua Scripting**
```python
from redis import asyncio as aioredis
from collections.abc import Callable

client = aioredis.from_url("redis://localhost")

# The Lua script runs atomically on the Redis server.
# KEYS[1]: The cache key
# ARGV[1]: The computed value to store if the key is missing
# ARGV[2]: The TTL in seconds
LUA_CACHE_SCRIPT = """
local cached_value = redis.call('get', KEYS[1])
if cached_value then
    return cached_value
else
    redis.call('set', KEYS[1], ARGV[1])
    redis.call('expire', KEYS[1], tonumber(ARGV[2]))
    return ARGV[1]
end
"""

async def get_or_compute(key: str, compute_func: Callable[[], str], ttl: int) -> str:
    """
    Check cache first; only compute on a miss, then store atomically via Lua.
    """
    # 1. Fast path: check the cache before doing expensive work
    cached = await client.get(key)
    if cached is not None:
        return cached.decode("utf-8") if isinstance(cached, bytes) else cached
    
    # 2. Cache miss: compute the value
    value = compute_func()
    
    # 3. Store atomically via Lua (prevents race with concurrent misses)
    result = await client.eval(LUA_CACHE_SCRIPT, 1, key, value, ttl)
    return result.decode("utf-8") if isinstance(result, bytes) else result
```