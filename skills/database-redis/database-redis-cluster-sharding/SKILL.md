---
name: database-redis-cluster-sharding

description: Guides the agent on Redis Cluster horizontal scaling — the 16,384 hash-slot model and CRC16 key-to-slot mapping, cluster configuration and creation (redis-cli --cluster create, --cluster-replicas), inspecting topology (CLUSTER NODES/SHARDS/SLOTS/KEYSLOT), and the operational limitations (no automatic rebalancing, predefined topology) worth knowing before committing to it.
---

# Redis Cluster (Sharding)

You are an expert in Redis Cluster. When a user needs to scale Redis horizontally across multiple nodes — beyond what a single instance's memory/CPU can serve — guide them to Redis Cluster's built-in sharding model, and make sure they understand the hash-slot mechanism that determines where any given key actually lives, plus the real operational limitations of managing it themselves.

- **Redis Cluster's scaling unit is the hash slot, not the key or the node directly.** The entire keyspace is divided into a fixed 16,384 slots; every node in the cluster owns some contiguous (or not) range of them. A 3-shard cluster commonly splits this as slots 0–5460 on shard 1, 5461–10921 on shard 2, and 10922–16383 on shard 3 — but the split doesn't have to be even or contiguous per node, that's just the common simple case.
- **Slot assignment for a given key is deterministic: `CRC16(key) mod 16384`.** This is why Redis Cluster can route any command to the correct shard without a lookup table for individual keys — the client (or the cluster itself, via redirection) computes which slot a key belongs to directly from the key's own bytes.
- **A node failure loses its slot range's data, unless that range is replicated.** This is the critical fact to internalize before treating Cluster mode as sufficient HA on its own: sharding by itself has no redundancy — losing a node with no replica for its slots means the data in those slots is gone until that node recovers. Always pair Cluster with `--cluster-replicas` (below) if losing a node should not mean losing data.
- **Enable clustering in the config, then create the topology explicitly — Redis doesn't auto-discover or auto-assign a cluster shape for you:**

  ```
  cluster-enabled yes
  cluster-config-file <filename>
  cluster-node-timeout <milliseconds>
  ```

  ```bash
  redis-cli --cluster create 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003
  ```

  This creates three independent, unreplicated shards — fine for pure horizontal scale with no redundancy requirement, but see the node-failure risk above before choosing this shape for anything where losing a shard's data is unacceptable.
- **Add `--cluster-replicas N` to get per-shard replication as part of cluster creation** — Redis Cluster decides which nodes become primaries and which become replicas for you from the node list given:

  ```bash
  redis-cli --cluster create 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 \
                              127.0.0.1:7004 127.0.0.1:7005 127.0.0.1:7006 \
                              --cluster-replicas 1
  ```

  This example yields 3 primary shards + 3 replicas (one per primary) — Cluster mode's own built-in answer to "what happens when a shard node fails," distinct from (and not layered on top of) Sentinel — see `database-redis-high-availability` for why Sentinel and Cluster are mutually exclusive HA strategies, not combinable.

## Inspecting Cluster Topology

- **`CLUSTER NODES`** — lists every node, its role (`master`/`slave`), and its owned slot range(s):

  ```
  redis-cli -p 7001 CLUSTER NODES
  ```

- **`CLUSTER SHARDS`** — a more structured, shard-centric view of the same information (slot ranges grouped with the nodes serving them).
- **`CLUSTER SLOTS`** — the slot-range-to-node mapping directly, useful for building client-side routing logic or just understanding the current layout at a glance.
- **`CLUSTER KEYSLOT <key>`** — compute which slot a specific key hashes to, without needing to run `CRC16` yourself:

  ```
  redis-cli -p 7001 CLUSTER KEYSLOT myname
  (integer) 12807
  ```

  Use this when debugging "why did this multi-key operation fail" — a cross-slot operation error almost always traces back to two keys landing in different slots; checking each key's slot directly confirms it rather than guessing.

## Known Limitations

Be upfront about these before committing to self-managed Cluster — they're real, ongoing operational costs, not one-time setup friction:

- **No automatic rebalancing/resharding when nodes are added or removed.** Growing or shrinking a cluster requires manually moving slot ranges (and their data) between nodes — Redis Cluster gives you the primitives for this, but doesn't decide or execute a rebalance on its own the way some managed/cloud offerings do. Budget real operational time for this whenever cluster topology changes are anticipated.
- **The node topology must be predefined at cluster-creation time** — you can't casually grow into a differently-shaped cluster without deliberate, manual slot migration.
- **Cross-slot multi-key operations aren't ordinarily supported** — a command touching keys in different slots is expected to fail in a standard cluster setup, by design (this is what data-locality-per-shard is built on). If a workload frequently needs multi-key atomicity/operations, deliberately colocate related keys into the same slot (see hash tags — `{same-tag}` substrings in key names force those keys into the same slot) rather than fighting the cluster's routing model after the fact.

## Code Example

```python
from redis.cluster import RedisCluster

# redis-py's cluster client handles slot routing/redirection transparently.
rc = RedisCluster(host="127.0.0.1", port=7001)

def keyslot_for(key: str) -> int:
    return rc.keyslot(key)

def set_and_get(key: str, value: str) -> str:
    rc.set(key, value)
    return rc.get(key)
```
