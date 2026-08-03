---
name: database-redis-leaderboards

description: Guides the agent on modeling leaderboards and rankings with Redis Sorted Sets — score-ordered range queries, top-N retrieval, rank lookup, and lexicographic ordering when scores tie — instead of sorting in application code.
---

# Redis Leaderboards (Sorted Sets)

You are an expert in modeling leaderboards, rankings, and scored collections with Redis. When a user needs to track scores for many participants and query by rank or score range, guide them to the Sorted Set data structure and its range/rank commands, instead of fetching all scores and sorting client-side.

- **A leaderboard is a Sorted Set: one `ZADD` per score update, with the member as the participant and the score as their current value.** The Sorted Set keeps members ordered automatically (backed by a skip list), so every subsequent query benefits from that ordering without re-sorting:

  ```
  ZADD user:score 234 John 232 Tim 1234 Dan 27 Eva 2213 Julia 32 Dylan 898 Molly
  ```

- **This is the right structure specifically because leaderboards are both write-heavy and read-heavy at once**: potentially millions of participants updating their score concurrently, and potentially millions of viewers polling for current rankings. A Sorted Set gives low-complexity operations for both sides — score updates and range reads — without needing a read-replica or a separate reporting store to keep the write path fast.
- **A Sorted Set's memory cost scales with the total number of distinct members ever added, not just the leaders** — for genuinely huge participant counts where only the current top-N is ever actually shown (and every participant's exact rank is never queried), `database-redis-topk` trades exact full-ranking for fixed, bounded memory. Default to a Sorted Set (exact, queryable at any rank); reach for Top-K specifically once member cardinality makes that memory cost a real problem and only the current leaders matter.
- **Retrieve a score range with `ZRANGE ... BYSCORE`**, not by fetching everything and filtering in the client:

  ```
  ZRANGE user:score 0 100 BYSCORE WITHSCORES
  ```

- **Retrieve the top (or bottom) N with `ZRANGE` using negative indices**, rather than fetching the whole set and taking a slice client-side:

  ```
  ZRANGE user:score -2 -1 WITHSCORES
  ```

  Negative indices count from the highest-scored end — `-1` is the top scorer, `-2` the second-highest, and so on. This is the idiomatic way to get "top N" without a separate sort step.
- **Get one participant's rank with `ZRANK`** (or `ZREVRANK` for descending rank) instead of computing position by counting through a fetched range:

  ```
  ZRANK user:score Madrid
  ```

- **When every score is equal (or omitted from ranking-relevance), a Sorted Set can still give ordered results — alphabetically — via `BYLEX`.** Set every member's score to the same value, then query lexicographic ranges:

  ```
  ZADD user:names 0 Tim 0 Dan 0 Eva 0 Julia 0 Dylan 0 Molly
  ZRANGE user:names [D "(D\xff" BYLEX
  ```

  Reach for this when the requirement is "alphabetical listing with range queries" (e.g. "names starting with D") rather than reaching for a separate Set plus client-side sorting — the same data structure covers both the score-ranked and the lexicographic case, just with different query modes.
- **Don't maintain the ranking by reading all scores into the application and sorting on every request.** That throws away the entire benefit of the Sorted Set's self-maintained order — every `ZADD` keeps the structure ordered incrementally, so a range/rank query is always O(log N + M), not a full re-sort of every participant on every read.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def update_score(leaderboard_key: str, player: str, score: float) -> None:
    await client.zadd(leaderboard_key, {player: score})

async def top_n(leaderboard_key: str, n: int) -> list[tuple[str, float]]:
    """Highest N scores, descending — negative indices select from the top end."""
    return await client.zrange(leaderboard_key, -n, -1, withscores=True, desc=True)

async def scores_between(leaderboard_key: str, min_score: float, max_score: float) -> list[tuple[str, float]]:
    return await client.zrangebyscore(leaderboard_key, min_score, max_score, withscores=True)

async def player_rank(leaderboard_key: str, player: str) -> int | None:
    """0-indexed rank, ascending by score; use zrevrank for descending rank."""
    return await client.zrank(leaderboard_key, player)
```
