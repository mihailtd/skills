---
name: database-redis-data-modeling-fundamentals

description: Guides the agent on mapping application objects and collections onto Redis's native data structures (Hash, Set, Sorted Set, List) instead of relational rows or opaque blobs, and on efficient primary-key access patterns.
---

# Redis Data Modeling Fundamentals

You are an expert in Redis data modeling. When a user is deciding how to represent an entity, collection, or relationship in Redis, guide them toward the native data structure that matches the shape of the data, instead of serializing everything into a single string value.

- **Persist your language's native structures directly, not a serialized blob.** Redis's core structures (Hash, Set, Sorted Set, List, Bitmap) mirror the data structures already used in application code (a dict/object → Hash, a list → List, a unique collection → Set). Storing data in the structure that matches its shape avoids the "impedance mismatch" of serializing to JSON/pickle just to get it into a single string key, and it unlocks structure-specific operations (partial field access, atomic increments, range queries) that a blob can't offer.
- **Model an entity as a Hash, keyed by a namespaced primary key.** Use a `type:id` naming convention (e.g. `city:653`, `user:2345`) so the key itself carries the entity type and primary key — this is the direct analogue of a relational table's primary key lookup, and it's the single fastest access pattern in Redis.
  - `HSET city:653 Name "Madrid" CountryCode "ESP" District "Madrid" Population 2879052` stores every field of the row in one Hash.
  - Prefer fetching only the fields actually needed (`HMGET`/`HGET`) over `HGETALL` when the object is wide and the caller only needs one or two attributes — this reduces bandwidth the same way `SELECT col1, col2` beats `SELECT *` in SQL.
- **Model a collection as a Set, Sorted Set, or List depending on what property of the collection actually matters:**
  - **Set** — unordered, unique membership. Use when you need "does X belong to this collection" (`SISMEMBER`, O(1)) or "give me all members" (`SMEMBERS`), with no ordering or duplicate requirement.
  - **Sorted Set** — unique membership *plus* a numeric score that keeps members ordered (backed by a skip list internally, giving low-complexity range queries). Use for leaderboards, rankings, or any case where you'll query "members above/below a threshold" (`ZRANGE ... BYSCORE`) or "this member's rank" (`ZRANK`) — not just membership.
  - **List** — ordered, allows duplicates, optimized for push/pop at either end. Use for queues, recent-activity feeds, or anywhere insertion order (not a score) defines the sequence.
  - Don't default to Set when a Sorted Set's ranking capability is actually needed later — retrofitting a score onto a plain Set means migrating the whole structure.
- **A single dictionary or list literal in your client language can usually be written to Redis without transformation** — e.g. `r.hset(f"user:{id}", mapping=user_dict)` for a Python dict, or `r.sadd("coding", *languages)` for a Python list — reinforcing that Redis modeling should start from "what native structure does this already look like in code," not "how do I serialize this."

## Code Examples

The same modeling choices — Hash for an entity, List for an ordered queue, Set for unique membership, Sorted Set for a ranked index — translate directly across client libraries, because they're decisions about Redis's data structures, not about any one language's idioms.

**Python (redis-py)**

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def save_city(city_id: int, name: str, country_code: str, district: str, population: int) -> None:
    """Model a relational row as a Hash, keyed by type:id."""
    await client.hset(
        f"city:{city_id}",
        mapping={"Name": name, "CountryCode": country_code, "District": district, "Population": population},
    )

async def get_city_summary(city_id: int) -> dict:
    """Fetch only the fields needed, instead of the whole Hash."""
    name, population = await client.hmget(f"city:{city_id}", ["Name", "Population"])
    return {"name": name, "population": population}

async def enqueue_and_dequeue(queue_key: str, *tasks: str) -> str | None:
    """List: ordered by insertion, push left / pop right for FIFO."""
    await client.lpush(queue_key, *tasks)
    return await client.rpop(queue_key)

async def dedupe_ids(set_key: str, *ids: str) -> set:
    """Set: unique membership, duplicates silently collapse."""
    await client.sadd(set_key, *ids)
    return await client.smembers(set_key)

async def index_by_population(scores: dict[str, int]) -> None:
    """Sorted Set: a ranked index, score = population."""
    await client.zadd("city:esp", scores)

async def top_cities_over(min_population: int) -> list[str]:
    """Low-complexity range query enabled by the Sorted Set's skip-list ordering."""
    return await client.zrangebyscore("city:esp", min_population, "+inf")
```

**JavaScript/TypeScript (node-redis)**

```typescript
import { createClient } from "redis";

const client = createClient({ url: "redis://localhost:6379" });
await client.connect();

async function saveCity(cityId: number, name: string, countryCode: string, population: number): Promise<void> {
  await client.hSet(`city:${cityId}`, { Name: name, CountryCode: countryCode, Population: population });
}

async function getCitySummary(cityId: number): Promise<Record<string, string>> {
  const [name, population] = await client.hmGet(`city:${cityId}`, ["Name", "Population"]);
  return { name, population };
}

async function enqueueAndDequeue(queueKey: string, ...tasks: string[]): Promise<string | null> {
  await client.lPush(queueKey, tasks);
  return client.rPop(queueKey);
}

async function dedupeIds(setKey: string, ...ids: string[]): Promise<string[]> {
  await client.sAdd(setKey, ids);
  return client.sMembers(setKey);
}

async function indexByPopulation(scores: { score: number; value: string }[]): Promise<void> {
  await client.zAdd("city:esp", scores);
}

async function topCitiesOver(minPopulation: number): Promise<string[]> {
  return client.zRangeByScore("city:esp", minPopulation, "+inf");
}
```

**Go (go-redis)**

```go
package main

import (
	"context"
	"strconv"

	"github.com/redis/go-redis/v9"
)

func saveCity(ctx context.Context, rdb *redis.Client, cityID int, name, countryCode string, population int) error {
	cityKey := "city:" + strconv.Itoa(cityID)
	return rdb.HSet(ctx, cityKey, map[string]interface{}{
		"Name": name, "CountryCode": countryCode, "Population": population,
	}).Err()
}

func getCitySummary(ctx context.Context, rdb *redis.Client, cityKey string) (map[string]string, error) {
	// Fetch only the fields needed, instead of the whole Hash.
	vals, err := rdb.HMGet(ctx, cityKey, "Name", "Population").Result()
	if err != nil {
		return nil, err
	}
	return map[string]string{"name": vals[0].(string), "population": vals[1].(string)}, nil
}

func enqueueAndDequeue(ctx context.Context, rdb *redis.Client, queueKey string, tasks ...string) (string, error) {
	if err := rdb.LPush(ctx, queueKey, tasks).Err(); err != nil {
		return "", err
	}
	return rdb.RPop(ctx, queueKey).Result()
}

func topCitiesOver(ctx context.Context, rdb *redis.Client, minPopulation float64) ([]string, error) {
	return rdb.ZRangeByScore(ctx, "city:esp", &redis.ZRangeBy{
		Min: strconv.FormatFloat(minPopulation, 'f', -1, 64), Max: "+inf",
	}).Result()
}
```
