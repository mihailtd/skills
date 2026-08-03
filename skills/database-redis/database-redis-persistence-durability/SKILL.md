---
name: database-redis-persistence-durability

description: Guides the agent on configuring Redis persistence for durability — AOF vs. RDB snapshots, the appendfsync policy tradeoff (no/everysec/always) with real throughput/latency numbers, the write(2)/fsync/OS-buffer-cache mechanics that explain what's actually at risk, and recovering a corrupted AOF with redis-check-aof — for deciding how much durability a use case actually needs to pay for.
---

# Redis Persistence and Durability

You are an expert in Redis persistence configuration. When a user needs to decide how durable a Redis deployment should be — from "pure cache, losing everything on restart is fine" to "primary database, near-zero data loss required" — guide them to the specific AOF/RDB configuration that matches the actual requirement, and make sure they understand the real, measured performance cost of the strongest settings before defaulting to them.

## What's Actually at Risk, and When

- **A write reaching Redis's in-memory keyspace is not yet safe from a crash** — understanding the path a write takes to actually become durable is what makes the `appendfsync` tradeoff legible rather than a mysterious dial:
  1. Redis calls `write(2)`, handing the data to the OS, which lands it in the **OS buffer cache** — not yet on the physical disk.
  2. Redis calls `fsync` (per whatever policy — see below), which forces the buffer cache's contents to the storage device's own cache.
  3. The disk controller persists from its cache to the physical medium.
  - **The OS buffer cache alone already protects against a Redis process crash or OOM-kill** — the data survived the `write(2)` call, and the OS will flush it to disk on its own (Linux does this roughly every 30 seconds by default) independent of whether Redis is still running. What `fsync` protects against specifically is a *system*-level failure (power loss, kernel panic, host crash) that takes the OS buffer cache down with it before that automatic flush happens.
- **This means the real question behind every `appendfsync` choice is: "how much data can I afford to lose if the whole machine dies at the worst possible moment?"** — not "will Redis crash," which the buffer cache already covers.

## AOF: appendfsync Policy

Enable with `appendonly yes`, then choose the fsync policy deliberately based on the answer to the question above — this is the single highest-leverage durability/performance tradeoff in Redis:

- **`appendfsync no`** — Redis never calls `fsync` itself; the OS flushes on its own schedule (~every 30s on Linux). Up to ~30 seconds of writes can be lost on a system-level failure. Fastest option; only appropriate when that loss window is genuinely acceptable.
- **`appendfsync everysec`** — Redis calls `fsync` once per second. Up to ~1 second of writes can be lost. This is the practical default for most workloads: a large majority of the durability of `always`, at a small fraction of the performance cost.
- **`appendfsync always`** — every write is flushed to disk before acknowledging it. The strongest guarantee (at most, the last *group* of commits in the current event-loop iteration can be lost, not more), but the cost is severe and needs to be measured, not assumed:

  | Policy | Throughput (SET, benchmark) | p50 latency | p99 latency |
  |---|---|---|---|
  | `everysec` | ~118,000 req/s | ~6.2 ms | ~40.5 ms |
  | `always` | ~15,000 req/s | ~56.3 ms | ~65.2 ms |

  (Illustrative numbers from a `redis-benchmark -t set -r 10000 -n 100000 -P 16` run — re-measure on your own hardware and workload shape before committing to a policy; don't treat these as universal constants.) That's roughly an 8x throughput drop and an order-of-magnitude latency increase going from `everysec` to `always` — a real cost that has to be justified by a real requirement, not applied reflexively as "the safe setting."
  - **Group commit softens this cost under concurrency**: when multiple writes land in the same event-loop iteration, Redis performs one `fsync` for the whole group rather than one per write, which is why `always` can still sustain hundreds of transactions/second rather than collapsing to single-digit throughput.
- **Decide using the actual shape of the workload, not a blanket policy**: read-heavy workloads can tolerate a more relaxed AOF policy since less is ever at risk; a workload already spread across Cluster shards gets parallel I/O bandwidth across multiple storage devices, changing the calculus; and if losing up to a second of recent writes has a real, bounded, acceptable cost (e.g. a session store, where an affected user just logs in again), `everysec` is very likely the right default rather than reaching for `always` out of caution.

## RDB Snapshots

- **`SAVE` (synchronous, blocks the server) / `BGSAVE` (asynchronous, forks to snapshot in the background) produce a point-in-time binary dump**, configured for periodic automatic collection:

  ```
  save 900 1000     -- snapshot if at least 1000 keys changed within 900 seconds
  dbfilename "dump.rdb"
  ```

- **RDB snapshots are not a substitute for AOF as a durability mechanism** — they're collected at intervals (commonly every few minutes to hours, since snapshotting has real performance cost), so the recovery point objective (RPO) they offer is measured in that interval, not seconds. Treat RDB as a backup/point-in-time-restore mechanism (copy the `.rdb` file to remote/external storage on a schedule) complementary to AOF, not a replacement for it when a tight RPO is the actual requirement.

## Recovering a Corrupted AOF

- **A crash timed exactly during an AOF write can leave a truncated/corrupted transaction on disk**, even under the strictest `appendfsync always` policy — Redis writes a transaction with a single `write(2)` call rather than a double-write/journaling scheme, so there's no automatic self-healing for a write that was interrupted mid-flight.
- **On restart, Redis detects this and refuses to start** rather than silently loading a corrupted/ambiguous state:

  ```
  # Unexpected end of file reading the append only file appendonly.aof.1.incr.aof.
  # You can: 1) Make a backup of your AOF file, then use
  # ./redis-check-aof --fix <filename.manifest>. 2) Alternatively you can set the
  # 'aof-load-truncated' configuration option to yes and restart the server.
  ```

- **Use `redis-check-aof --fix <manifest>` to repair it** — this truncates the file back to the last complete, valid transaction, discarding the incomplete one, so the server can restart cleanly:

  ```bash
  redis-check-aof --fix appendonlydir/appendonly.aof.manifest
  ```

  **Back up the AOF file before running `--fix`** — the tool is explicit about how many bytes it's about to discard and asks for confirmation, but the discarded data is genuinely gone once applied; keep a copy in case anything about the truncation needs to be inspected afterward.
- **`aof-load-truncated` controls whether Redis auto-recovers from a truncated AOF at startup instead of refusing to start** — `yes` (the Redis 7 default) lets the server come up with the truncated file as-is; `no` forces the explicit `redis-check-aof --fix` workflow above. Setting it to `no` is a reasonable choice when you want a human to consciously acknowledge and inspect data loss rather than have it happen silently on every restart.
- **A battery-backed RAID write-caching controller adds a hardware-level safety net on top of `fsync`**: the controller can safely complete a flush from its own battery-backed cache to disk even through a power loss, because the battery keeps that cache alive long enough to finish. This is a genuine additional layer of protection, not a substitute for choosing the right `appendfsync` policy — the two operate at different levels (application-level fsync discipline vs. hardware-level write safety).

## Pipelining Doesn't Weaken Durability Guarantees

- **Clients using pipelining trade per-command acknowledgment for throughput, not durability** — writes and fsyncs (per whatever policy is configured) still happen at the end of each event-loop iteration regardless of whether the client used pipelining. Don't assume pipelined writes are "less safe" than one-at-a-time writes under the same `appendfsync` policy; the persistence guarantee is unchanged, only the client's visibility into individual command results is different.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def check_persistence_config() -> dict:
    """Inspect the active durability settings before assuming what's configured."""
    appendonly = await client.config_get("appendonly")
    appendfsync = await client.config_get("appendfsync")
    save = await client.config_get("save")
    return {**appendonly, **appendfsync, **save}

async def set_conservative_durability() -> None:
    """A reasonable default for most workloads: strong-enough durability, sustainable throughput."""
    await client.config_set("appendonly", "yes")
    await client.config_set("appendfsync", "everysec")

async def trigger_background_snapshot() -> None:
    """Point-in-time backup — complementary to AOF, not a replacement for it."""
    await client.bgsave()
```
