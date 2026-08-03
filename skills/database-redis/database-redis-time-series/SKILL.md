---
name: database-redis-time-series

description: Guides the agent on modeling and querying time-stamped numeric data with Redis Stack's time series type — TS.CREATE tuning (RETENTION, ENCODING, CHUNK_SIZE, DUPLICATE_POLICY), atomic counters with TS.INCRBY/DECRBY, label-based multi-series queries (TS.MRANGE/TS.MGET/TS.QUERYINDEX), the full aggregation function set, and TS.CREATERULE compaction (including the retroactivity gotcha) — for monitoring, sensor, and metrics use cases.
---

# Redis Stack Time Series

You are an expert in Redis Stack's time series data type. When a user needs to store and query data points that occur at specific points in time (metrics, sensor readings, monitoring events), guide them to the native time series type instead of modeling timestamps as a Sorted Set score or hand-rolling time-bucketed keys.

- **Use the time series type whenever the core question is "what was the value of X over this time range," not just "what is the current value of X."** A single counter or gauge stored as a plain string/Hash field only ever tells you the latest value; a time series retains the sequence of samples so range queries, aggregation, and trend analysis are possible later.
- **Create the series explicitly before adding samples**, with a key name that identifies what's being measured. The bare form is fine to start, but every tuning parameter below is worth considering deliberately rather than accepting every default:

  ```
  TS.CREATE app:monitor:temp
  TS.CREATE app:monitor:temp2 RETENTION 604800000 ENCODING COMPRESSED CHUNK_SIZE 4096 DUPLICATE_POLICY LAST LABELS device sensor1 unit celsius
  ```

  - **`RETENTION`** (milliseconds) — how long a sample survives before it expires, measured from its own timestamp forward, checked lazily on subsequent `TS.ADD`/`TS.MADD`/`TS.INCRBY`/`TS.DECRBY` calls to the same key (not a background sweep). Default `0` means samples never expire — fine for a metric you'll always want full history for, but a real memory-growth risk for high-frequency data with no compaction/retention plan. Decide retention deliberately for anything ingested continuously.
  - **`ENCODING`** — `COMPRESSED` (default, using the lossless Gorilla algorithm) typically cuts memory usage dramatically (up to ~90%) and improves performance by reducing memory accesses; `UNCOMPRESSED` stores raw samples and is only worth choosing when timestamps/values are highly irregular and infrequent enough that compression doesn't help. Default to `COMPRESSED` unless you've measured a specific reason not to.
  - **`CHUNK_SIZE`** (bytes, multiple of 8, 48–1,048,576, default 4096) — the storage-allocation unit per chunk. Smaller chunks slow down inserts (more chunks to allocate/manage) but speed up small-range queries; larger chunks help insert throughput but make small-range queries slower (more of a chunk to scan past). Changing it later via `TS.ALTER` only affects *new* chunks, not existing ones — so it's much cheaper to pick correctly upfront based on expected ingest rate and typical query range than to rely on being able to fix it later.
  - **`DUPLICATE_POLICY`** — what happens when a new sample arrives with a timestamp that already has a value: `BLOCK` (default; reject with an error), `FIRST` (keep the existing value, silently ignore the new one), `LAST` (overwrite), `MIN`/`MAX` (keep whichever value is smaller/larger), `SUM` (add the new value to the existing one). The default (`BLOCK`) is the safest choice when duplicate timestamps genuinely shouldn't happen and should be surfaced as an error; pick `LAST` for a "most recent value wins" sensor feed, or `SUM` for a counter that might receive multiple partial increments landing on the same millisecond.
- **Add samples with `TS.ADD`, letting the server assign the timestamp unless you have one from the source** (`*` for "now," in server time):

  ```
  TS.ADD app:monitor:temp * 20
  TS.ADD app:monitor:temp * 20.1
  ```

  Each call returns the millisecond Unix timestamp actually stored — capture it if the caller needs to correlate the sample with the exact time it was recorded, rather than re-deriving it separately.
- **When the series is really a counter (visit counts, event tallies), use `TS.INCRBY`/`TS.DECRBY` instead of a manual read-increment-write cycle.** These apply the increment/decrement atomically server-side and return the insertion timestamp, removing an entire class of race condition that a client-side "read current value, add to it, write it back" approach would have under concurrent writers:

  ```
  TS.INCRBY mortensi.com 1
  TS.INCRBY mortensi.com 1
  ```

  Each call adds a new sample (not merely mutating an existing one), so the full history of increments — not just the running total — remains queryable.
- **Get the most recent sample directly with `TS.GET`** — no need to fetch a range and take the last element:

  ```
  TS.GET mortensi.com
  ```

- **Query a time range with `TS.RANGE`**, bounded by two timestamps (or `-`/`+` for "from the beginning"/"to the latest"), instead of fetching every sample and filtering client-side:

  ```
  TS.RANGE app:monitor:temp 1675632818179 1675632829519
  TS.RANGE mortensi.com - +
  ```

  This returns exactly the samples in that window, in time order, with no need to know how many samples exist or scan past irrelevant ones.
- **Don't reach for a Sorted Set with a timestamp as the score as a substitute for time series** once the data is genuinely a metric stream — a Sorted Set gives ordering and range queries by score, but the time series type additionally understands the *semantics* of time-stamped numeric data (retention policies, downsampling/compaction rules, and aggregation over time buckets), none of which a plain Sorted Set provides natively.
- **Downsample automatically with a compaction rule instead of aggregating on every read.** `TS.CREATERULE` links a raw source series to a destination series that the server keeps continuously updated with a rolling aggregate (`avg`, `sum`, `min`, `max`, `range`, `count`, `first`, `last`, and more) over a fixed time bucket:

  ```
  TS.CREATE app:34:cpu
  TS.CREATE app:34:cpu:max
  TS.CREATERULE app:34:cpu app:34:cpu:max AGGREGATION max 10000
  ```

  Every `TS.ADD` to the source series (`app:34:cpu`) automatically updates the derived series (`app:34:cpu:max`) — the destination series is queryable with a plain `TS.RANGE` and already contains one point per 10-second bucket, with no client-side aggregation logic and no re-scanning of raw samples on every read:

  ```
  TS.RANGE app:34:cpu:max - +
  ```

  Reach for a compaction rule whenever a dashboard or alert only ever needs a bucketed aggregate (e.g. "max CPU per 10 seconds," not every raw sample) — querying the raw series and aggregating per request repeats work that a rule computes once, incrementally, as data arrives. Pick the destination bucket size and aggregation function to match how the data will actually be consumed (alerting on spikes wants `max`; capacity planning wants `avg`), not a single default applied everywhere.
  - **`TS.CREATERULE` is not retroactive.** Aggregation only applies to samples added to the source series *after* the rule was created — pre-existing samples in the source series are never backfilled into the destination series. If historical data needs to appear in the downsampled series too, that requires a separate, explicit backfill step (e.g. re-running the aggregation manually over the old range) — don't assume creating a rule on a series that already has data will populate the destination series with anything beyond what arrives from that point forward.
  - **The destination series must exist (via its own `TS.CREATE`) before `TS.CREATERULE` links it** — the rule doesn't implicitly create it.

## Labels: Metadata for Multi-Series Queries

- **Labels are key-value metadata attached to a series** (at creation via `LABELS`, or added/changed later via `TS.ALTER ... LABELS`) — not data points themselves, but a secondary index over the *series*, letting you group/filter/query across many series by shared attributes instead of tracking key names by convention:

  ```
  TS.ALTER mortensi.com LABELS dev python database redis
  ```

- **Query multiple series at once by label with `TS.MRANGE`/`TS.MREVRANGE`/`TS.MGET`**, instead of issuing one `TS.RANGE`/`TS.GET` per key and merging results in application code:

  ```
  TS.MRANGE - + FILTER database=redis
  ```

  This is the direct multi-series analogue of `TS.RANGE`/`TS.GET` — same time-range/latest-value semantics, applied across every series matching the label filter in one call. Use it whenever a query is naturally "all series with property X," not "this one specific series I already know the key for."
- **Discover which series match a label filter with `TS.QUERYINDEX`**, without fetching any actual data points — useful for building a dynamic list of available series (e.g. populating a dashboard's series picker) without hardcoding key names.
- **Design labels around the dimensions you'll actually query/group by** (data source, unit, device/location, environment) — a label scheme decided upfront makes `TS.MRANGE ... FILTER` queries and cross-series comparisons trivial; retrofitting labels onto many already-created series (one `TS.ALTER` call each) is more work than deciding the scheme before ingesting data.

## Aggregation Functions

The same function names apply both to ad hoc range aggregation (`TS.RANGE`/`TS.MRANGE` with an `AGGREGATION` clause) and to a `TS.CREATERULE` compaction rule — pick based on what the resulting number should mean, not by habit:

| Function | Meaning |
|---|---|
| `avg` | Mean value in the bucket |
| `sum` | Total of values in the bucket |
| `min` / `max` | Lowest / highest value in the bucket |
| `range` | `max - min` within the bucket — a quick spread/volatility indicator |
| `count` | Number of samples in the bucket — useful for spotting gaps in ingestion, not just the metric itself |
| `first` / `last` | The bucket's earliest / most recent sample — useful for "current state per bucket" rather than a computed statistic |
| `std.p` / `std.s` | Population / sample standard deviation — spread of values within the bucket |
| `var.p` / `var.s` | Population / sample variance — the same spread measure `std.*` reports, unsquared |
| `twa` | Time-weighted average — accounts for uneven gaps between samples, unlike a plain `avg`, which treats every sample as equally spaced regardless of actual timing |

Prefer `twa` over `avg` specifically when samples arrive at irregular intervals and a straight arithmetic mean would over-weight periods with more (possibly redundant) samples.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def create_sensor_series(sensor_key: str) -> None:
    await client.ts().create(sensor_key)

async def record_reading(sensor_key: str, value: float) -> int:
    """Let the server timestamp the sample; returns the ms Unix timestamp used."""
    return await client.ts().add(sensor_key, "*", value)

async def readings_in_window(sensor_key: str, from_ts_ms: int, to_ts_ms: int) -> list[tuple[int, float]]:
    """Range query returns exactly the samples in the window, in time order."""
    return await client.ts().range(sensor_key, from_ts_ms, to_ts_ms)

async def create_max_per_10s_rollup(source_key: str, dest_key: str) -> None:
    """Server-maintained downsampled series: no client-side aggregation needed."""
    await client.ts().create(dest_key)
    await client.ts().createrule(source_key, dest_key, "max", bucket_size_msec=10_000)

async def max_cpu_per_bucket(dest_key: str) -> list[tuple[int, float]]:
    """Query the pre-aggregated series directly — already bucketed by the rule."""
    return await client.ts().range(dest_key, "-", "+")

async def create_tuned_series(key: str, retention_ms: int, labels: dict) -> None:
    """A production series: bounded retention, labeled for multi-series queries."""
    await client.ts().create(
        key, retention_msecs=retention_ms, encoding="COMPRESSED",
        duplicate_policy="LAST", labels=labels,
    )

async def increment_visit_counter(site_key: str) -> int:
    """Atomic counter increment — no read-modify-write race under concurrent writers."""
    return await client.ts().incrby(site_key, 1)

async def latest_value(key: str) -> tuple[int, float] | None:
    return await client.ts().get(key)

async def all_series_for_label(label_filter: str) -> dict:
    """Multi-series range query: every series matching the label filter, in one call."""
    return await client.ts().mrange("-", "+", filters=[label_filter])
```
