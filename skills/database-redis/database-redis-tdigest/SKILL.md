---
name: database-redis-tdigest

description: Guides the agent on Redis's t-digest structure (TDIGEST.CREATE/ADD/QUANTILE/CDF/RANK) for estimating quantiles/percentiles over a data stream in a compact sketch — "what value is the Nth percentile," "what fraction of values falls below X" — for latency distributions, scoring thresholds, and anomaly detection.
---

# Redis t-digest

You are an expert in Redis's t-digest probabilistic data structure. When a user needs to answer a question about the *distribution* of a stream of numeric values — percentiles, quantiles, "what fraction of values is below/above X" — guide them to t-digest instead of trying to compute this by sorting a large stored dataset on demand.

- **Recognize the question shape this structure answers, and the two directions it can be queried from:**
  - "What value marks the Nth percentile/quantile?" → `TDIGEST.QUANTILE` — give a fraction, get a value.
  - "What fraction/rank of observed values falls at or below this value?" → `TDIGEST.CDF` (fraction) / `TDIGEST.RANK` (count) — give a value, get a fraction or count.
  - These are inverses of each other; pick based on which side of the question you actually have (a target percentile in hand → `QUANTILE`; a specific value in hand and you want to know its standing → `CDF`/`RANK`).
- **Quantiles and percentiles are the same underlying concept at different granularity** — a percentile divides data into 100 equal-frequency parts (the 75th percentile: 75% of data falls below it); a quantile generalizes this to any number of divisions (the median, the 0.5 quantile, is a two-part case). `TDIGEST.QUANTILE` takes a fraction (0–1), so the "75th percentile" is expressed as `TDIGEST.QUANTILE key 0.75`.
- **Create, then stream values into the digest as they occur — no need to retain the raw values for later analysis:**

  ```
  TDIGEST.CREATE temperatures
  TDIGEST.ADD temperatures 20 43 38 24 41
  ```

  This is the core value proposition: the digest maintains enough structure to answer quantile/rank questions accurately without storing every observation — memory stays compact (a "sketch," not a full history) regardless of how many values have streamed through.
- **Query a quantile (value at a given fraction) with `TDIGEST.QUANTILE`:**

  ```
  TDIGEST.QUANTILE temperatures 0.5   -- median
  TDIGEST.QUANTILE temperatures 0.9   -- 90th percentile
  ```

- **Query the cumulative distribution (fraction of observations ≤ a value) with `TDIGEST.CDF`, or the count with `TDIGEST.RANK`:**

  ```
  TDIGEST.CDF temperatures 38    -- fraction of values <= 38
  TDIGEST.RANK temperatures 38   -- count of values <= 38
  ```

  Use `CDF` when the answer needs to be a proportion/percentage ("what % of requests were under 200ms"); use `RANK` when the answer needs to be a raw count ("how many of the last N requests were under 200ms").
- **Accuracy is best in the tails (extreme percentiles) and comparatively coarser near the median** — this is a deliberate property of the underlying algorithm (it allocates more precision to the distribution's edges), which happens to align well with the most common real use case: extreme percentiles (p95, p99) usually matter more for SLAs/alerting than the median does.
- **Use cases**: network/service latency distributions (what fraction of requests exceed an SLA threshold, pinpointing tail latency for troubleshooting), determining score thresholds in gaming/ranking ("what score puts a player in the top 5%"), and anomaly/DoS detection (is the latest burst of traffic outside the normal observed percentile range for a given time window). Any "where does X rank relative to everything we've seen" or "what threshold captures the top/bottom N%" question is a strong signal for t-digest.
- **This skill covers the core commands — t-digest supports additional operations (e.g. merging digests, trimmed mean) beyond what's shown here.** Confirm current command availability/syntax against the Redis Stack documentation for anything beyond basic create/add/quantile/CDF/rank.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def create_digest(key: str) -> None:
    await client.tdigest().create(key)

async def record_observations(key: str, *values: float) -> None:
    await client.tdigest().add(key, list(values))

async def percentile(key: str, fraction: float) -> float:
    """e.g. percentile(key, 0.95) for p95 latency."""
    result = await client.tdigest().quantile(key, fraction)
    return result[0]

async def fraction_at_or_below(key: str, value: float) -> float:
    result = await client.tdigest().cdf(key, value)
    return result[0]
```
