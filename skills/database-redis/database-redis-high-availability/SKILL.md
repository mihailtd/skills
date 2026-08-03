---
name: database-redis-high-availability

description: Guides the agent on configuring Redis high availability via replication (replicaof, min-replicas-to-write/min-replicas-max-lag) and Redis Sentinel-managed automatic failover — Sentinel's odd-quorum requirement, minimum process topology, and the operational limitations (manual node isolation, no geo-distributed DR/BC) worth knowing before committing to self-managed HA.
---

# Redis High Availability (Replication + Sentinel)

You are an expert in configuring Redis for high availability. When a user needs a Redis deployment to survive a primary instance failure without manual intervention, guide them through replication configuration, the write-safety knobs that bound acceptable data loss, and Sentinel's automatic-failover mechanism — while being upfront about what self-managed HA still doesn't cover.

- **Replication alone is not HA — it's the prerequisite for it.** A replica following a primary via `replicaof <masterip> <masterport>` gives you a live, near-current copy of the data, but if the primary dies, the replica just sits there unpromoted and inconsistent-looking on its own; nothing about plain replication makes a replica *become* the new primary automatically. That promotion step is what Sentinel (below) actually provides.
- **Basic replication configuration:**

  ```
  replicaof <masterip> <masterport>
  masterauth <master-password>
  masteruser <username>
  replica-serve-stale-data yes
  replica-read-only yes
  ```

  `replica-read-only yes` is the safe default — it prevents accidental writes landing on a replica instead of the primary, which would silently diverge from what the primary (and its other replicas) has.
- **Replication is asynchronous, so a replica can lag** — how far behind depends on write throughput, network bandwidth, and replica host resources (see `database-redis-replication-consistency` for the full mechanics of why, and for `WAIT`/`WAITAOF` as a per-write tightening of this guarantee). HA configuration and per-write consistency tightening are complementary, not substitutes for each other — Sentinel handles "what happens when the primary dies," `WAIT`/`WAITAOF` handles "how sure am I this specific write survived that."
- **Bound how stale a replica is allowed to be before writes get rejected, using `min-replicas-to-write`/`min-replicas-max-lag`:**

  ```
  min-replicas-to-write 1
  min-replicas-max-lag 10
  ```

  With this configured, the primary refuses new writes (returning an error to the client) unless at least the specified number of replicas have acknowledged within the specified lag window. This trades some write availability for a bound on how much data could be lost if the primary fails right after — deliberately reject writes rather than accept ones that might vanish on failover. Choose the numbers based on how much staleness/write-rejection risk is actually acceptable for the workload; there's no universally correct default.

## Sentinel: Automatic Failover

- **Sentinel is a separate Redis process, in a special monitoring mode, whose job is to detect a primary failure and promote a replica** — it doesn't store your application's data itself. Start it either via the Redis Stack binary with a Sentinel-specific config, or the dedicated binary:

  ```bash
  redis-stack-server /path/to/sentinel.conf --sentinel
  # or
  redis-sentinel /path/to/sentinel.conf
  ```

- **Sentinel configuration monitors a named primary and defines failure/failover thresholds:**

  ```
  sentinel monitor mymaster 127.0.0.1 6379 2
  sentinel down-after-milliseconds mymaster 60000
  sentinel failover-timeout mymaster 180000
  sentinel parallel-syncs mymaster 1
  ```

  The `2` in `sentinel monitor` is the quorum — how many Sentinel processes must agree the primary is actually down before triggering a failover, a safeguard against one Sentinel's own network hiccup falsely triggering a promotion.
- **Sentinel needs an odd number of instances, minimum three, specifically so a quorum-based election can never tie.** This is a hard requirement of the algorithm, not a suggestion — an even number (especially two) can deadlock on which Sentinel gets to decide.
- **The Sentinel count and the data-shard count are independent** — you don't need three primary shards to get Sentinel's benefit. A minimal HA setup is one primary, one replica, and three Sentinel processes: **five Redis processes total**, ideally each on its own host/server so a single host failure can't simultaneously take out a data shard and enough Sentinels to lose quorum. Skimping on this isolation (e.g. co-locating a Sentinel with the primary it's supposed to detect the failure of) undermines the whole point.
- **When the primary fails, Sentinel detects it (once quorum agrees) and promotes a replica to primary automatically** — this is the actual "high availability" outcome; replication by itself only gets you a warm standby, not an automatic one.

## Known Limitations of Self-Managed HA

Be upfront about these before committing to a self-managed Sentinel deployment — they're real operational costs, not solved by more careful configuration:

- **No automatic geographic distribution.** Sentinel-managed HA is a single-region mechanism; it doesn't address disaster recovery or business continuity patterns (Active-Passive/Active-Active across regions) on its own. If regional-outage resilience is a real requirement, that needs to be solved at a layer above plain Sentinel.
- **Real infrastructure cost**: isolating every Sentinel and data shard onto its own host (as recommended above) means at least five separate hosts/processes for even the minimal topology — a genuine cost to budget for, not a paper configuration exercise.
- **Sentinel and Cluster mode are mutually exclusive configurations** — you choose one HA/scaling strategy or the other, not both layered together. If the priority is *only* availability (not horizontal scale), Sentinel is the simpler, more directly-applicable choice; if scale/throughput is the actual driver, Cluster mode (see `database-redis-cluster-sharding`) comes with its own HA via per-shard replication, making a separate Sentinel deployment unnecessary on top of it.

## Code Example

```python
from redis import asyncio as aioredis
from redis.asyncio.sentinel import Sentinel

# Application code talks to Sentinel, not directly to a hardcoded primary address —
# Sentinel resolves the current primary, including across a failover.
sentinel = Sentinel([("sentinel1", 26379), ("sentinel2", 26379), ("sentinel3", 26379)])

async def get_primary_client():
    return sentinel.master_for("mymaster", socket_timeout=0.5)

async def get_replica_client():
    return sentinel.slave_for("mymaster", socket_timeout=0.5)
```
