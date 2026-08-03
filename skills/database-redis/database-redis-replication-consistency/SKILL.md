---
name: database-redis-replication-consistency

description: Guides the agent on Redis's asynchronous replication model — why a replica can lag behind its master, what that means for data loss on failover, and the WAIT/WAITAOF commands for tightening the consistency guarantee at a real, measurable throughput cost — for deciding how much replication consistency a use case actually needs.
---

# Redis Replication Consistency

You are an expert in Redis's replication consistency model. When a user needs to understand what happens to recent writes during a failover, or needs a stronger guarantee that a write has actually reached replicas before considering it "done," guide them to `WAIT`/`WAITAOF` and make sure they understand the asynchronous-by-default replication sequence those commands are compensating for.

- **Replication is asynchronous by default, and the exact sequence explains why a replica can be behind:**
  1. A client sends a write to the master (or the owning shard, in a clustered deployment).
  2. The master executes it and persists it locally per its own persistence policy (see `database-redis-persistence-durability`).
  3. **The client is acknowledged at this point** — before the write has been sent to any replica.
  4. Only then does the write enter a replication buffer and get propagated to replicas.
  - This ordering is what makes replication fast (the client never waits on network round trips to replicas), but it's also exactly why a replica can be measurably behind the master — how far behind depends on write throughput, network bandwidth, and replica host resources, not a fixed bound.
- **The consequence that actually matters: a replica promoted during a master failure may not have the most recent writes.** If the master crashes before a recent write propagated to the replica that gets promoted, that write is gone — not delayed, gone. Whether this is acceptable is entirely use-case dependent: losing a handful of auth tokens on failover just means affected users re-authenticate; losing a financial transaction record is a different order of problem. Make this evaluation deliberately per use case rather than assuming either "replication means I'm safe" or "replication is unsafe, don't use Redis as primary."
- **`WAIT numreplicas timeout` blocks until at least `numreplicas` replicas have acknowledged replicating everything written before the call** — use it after a write (or batch of writes) whose loss on an immediate failover would be a real problem, to get a much stronger (but not absolute) guarantee before considering the operation truly done:

  ```
  SET critical:key value
  WAIT 1 1000   -- block up to 1000ms for at least 1 replica to have replicated it
  ```

  **`WAIT` does not make Redis a strongly consistent store on its own** — synchronous replication is one ingredient of that, not the whole recipe. It meaningfully improves real-world data safety around Sentinel/Cluster failover specifically, which is the scenario it's designed for.
- **`WAIT` can produce false negatives — a timeout doesn't prove the data wasn't replicated, only that acknowledgment didn't arrive in time** (e.g. a transient network hiccup between master and replica). **Never treat a `WAIT` failure as certain proof of a lost write** — the application has to handle this ambiguity explicitly: check replica state, or re-run the operation if it's naturally idempotent (`SADD`, `ZADD`, `HSET`, and similar "set to this value" commands are safe to retry; a blind `INCR` is not, since retrying it double-counts).
- **`WAITAOF numlocal numreplicas timeout` extends the same idea to fsync, not just network replication** — it additionally confirms the local copy and/or the specified number of replicas have `fsync`ed the write to their own AOF, not merely received it over the wire:

  ```
  WAITAOF 1 1 1000   -- local AOF fsync + at least 1 replica's AOF fsync
  ```

  Combine `WAITAOF` with `appendfsync always` on both master and replica for the strongest achievable combination of durability and replication consistency Redis offers — and go in with clear eyes that this stacks the performance cost of strict AOF fsyncing (see `database-redis-persistence-durability`) with the added latency of waiting on replica acknowledgment. This is the right call for a narrow set of genuinely critical writes, not a sensible default for every write in a high-throughput workload.
- **Don't reach for `WAIT`/`WAITAOF` on every write by reflex.** They add real per-call latency (bounded by the `timeout` you specify, and by actual network/replica performance below that). Use them selectively, on the specific operations where losing the write on an immediate failover would be a real, unacceptable cost — for everything else, the default asynchronous replication (backed by whatever `appendfsync` policy is already configured) is the appropriate, faster choice.
- **High-availability failover mechanisms themselves (Sentinel-managed automatic failover, Redis Cluster's built-in replica promotion, Enterprise/Cloud's managed failover) are a deployment-topology decision, not a consistency-setting one** — `WAIT`/`WAITAOF` are what you reach for to tighten the *data* guarantee during whichever failover mechanism is in place, not a replacement for choosing and configuring one. See `database-redis-high-availability` for actually configuring replication and Sentinel-managed automatic failover, and `database-redis-cluster-sharding` for Cluster's built-in per-shard replication.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def write_with_replication_guarantee(key: str, value: str, min_replicas: int = 1, timeout_ms: int = 1000) -> bool:
    """Stronger guarantee for a write whose loss on failover would be a real problem."""
    await client.set(key, value)
    acked = await client.execute_command("WAIT", min_replicas, timeout_ms)
    return acked >= min_replicas  # a False here is not proof of loss — see the note on false negatives

async def write_with_fsync_and_replication_guarantee(key: str, value: str) -> list:
    """Strongest achievable guarantee: local + replica AOF fsync, not just network replication."""
    await client.set(key, value)
    return await client.execute_command("WAITAOF", 1, 1, 1000)
```
