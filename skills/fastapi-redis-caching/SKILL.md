---
name: fastapi-redis-caching
description: Instructs the agent on implementing the Cache-Aside pattern in FastAPI using Redis, both manually for complex logic and declaratively using the fastapi-cache library.
---

# FastAPI Caching & Cache-Aside Pattern Guidelines

You are an expert Python developer specializing in FastAPI and high-performance API design. When asked to implement caching, optimize endpoint performance, or integrate Redis into a FastAPI application, you must adhere to the following rules:

## 1. The Cache-Aside Pattern (Lazy Loading)
When implementing manual caching for specific data retrieval operations, you must strictly follow the Cache-Aside pattern [1, 3].
*   **Check Cache First:** The application must first query Redis using a specific, deterministic cache key [3, 4].
*   **Cache Hit:** If the data is present, deserialize it (e.g., from JSON) and return it immediately to the user without querying the primary database [3].
*   **Cache Miss:** If the data is absent, query the primary database (e.g., PostgreSQL, MongoDB, or Elasticsearch), store the serialized result in Redis, and then return the data [3, 5].

## 2. Mandatory TTL (Time-To-Live)
Never cache API responses or dynamic database queries without an expiration time [5]. 
*   When setting the value in Redis after a cache miss, you must use the `ex` parameter (e.g., `ex=3600`) to define the TTL in seconds [5]. This prevents memory bloat and ensures stale data is eventually refreshed.

## 3. Declarative Caching with `fastapi-cache`
For standard endpoint caching where the entire response can be cached, prioritize using the `fastapi-cache` library rather than writing manual Redis queries [2].
*   The `fastapi-cache` library abstracts the cache-aside pattern into simple endpoint decorators [2].
*   Use the `@cache` decorator on your FastAPI route functions to easily define the TTL (`expire`), backend (Redis or in-memory), and encoding [2].

---

## Code Examples

Below are best-practice examples demonstrating how to implement caching in FastAPI.

**1. Declarative Endpoint Caching (`fastapi-cache`)**
Use this approach for clean, readable endpoint caching [2].

```python
from fastapi import APIRouter
from fastapi_cache.decorator import cache
from typing import Any

router = APIRouter()

# The @cache decorator abstracts the entire cache-aside flow
@router.get("/top/ten/artists/{country}")
@cache(expire=3600)  # Cache the response for 1 hour
async def top_ten_artist_by_country(country: str) -> dict[str, Any]:
    # This block only executes on a cache miss
    data = await fetch_heavy_data_from_primary_db(country)
    return data
```

**2. Manual Cache-Aside Implementation (redis.asyncio)**
Use this approach when you need fine-grained control over what is cached, or when caching specific computations inside a larger endpoint.

```python
import json
import logging
from fastapi import APIRouter, Depends, HTTPException
from redis import asyncio as aioredis

logger = logging.getLogger(__name__)
router = APIRouter()

# Assuming a dependency that returns an aioredis client
async def get_redis_client() -> aioredis.Redis:
    return aioredis.from_url("redis://localhost")

@router.get("/custom/stats/{region}")
async def get_region_stats(
    region: str,
    redis_client: aioredis.Redis = Depends(get_redis_client)
):
    cache_key = f"stats:region:{region}"
    
    # 1. Query Redis first
    cached_data = await redis_client.get(cache_key)
    
    # 2. Cache Hit: Return data immediately
    if cached_data:
        logger.info(f"Returning cached data for {region}")
        return json.loads(cached_data)
        
    # 3. Cache Miss: Query the primary database
    try:
        stats = await execute_expensive_db_query(region)
    except Exception as e:
        raise HTTPException(status_code=400, detail="Database query failed")
        
    # 4. Store the result in Redis with an expiration time (TTL)
    await redis_client.set(cache_key, json.dumps(stats), ex=3600)
    
    return stats
```