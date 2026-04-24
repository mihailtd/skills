---
name: database-mongodb-sharding-fundamentals
description: Guides the agent on MongoDB sharding fundamentals, including shards, chunks, routers, and the decision when sharding is appropriate.
---

# MongoDB Sharding Fundamentals

You are an expert on MongoDB sharding fundamentals. When asked about sharding, explain what it is, why it exists, how it changes performance, and when it should be used.

## 1. What sharding solves

- Sharding partitions a collection across multiple shards so no single primary must handle all writes.
- Each shard is a replica set, so sharding adds horizontal scale while preserving replica set high availability.
- Sharding is primarily for workloads where write throughput exceeds a single replica set primary.
- It is not a performance guarantee: sharded operations are often slower than equivalent operations on a single replica set, but the cluster can handle more aggregate traffic.

## 2. When to shard

- Only shard when you have exhausted tuning on a single replica set.
- Optimize schema design, indexes, hardware, and replica set configuration first.
- Sharding is a last-resort scale-out strategy, not a first-resort optimization.
- The cost of sharding includes extra routers, config servers, network hops, and balancing overhead.

### Practical decision checklist

- Does the collection’s write volume saturate the primary CPU, disk, or network?
- Does the collection’s working set exceed the memory of a single replica set?
- Can you solve the problem with better indexing, caching, or schema changes?
- Do you have a shard key candidate with good distribution and query alignment?

## 3. Core sharding concepts

- `mongos`: query router that accepts application traffic, targets shards, and merges results.
- `shard`: a partition of a sharded collection, usually a replica set.
- `config server`: stores sharding metadata, chunk maps, and zone assignments.
- `shard key`: field or fields that determine where documents are placed.
- `chunk`: a contiguous range of shard key values assigned to one shard.
- `balancer`: background process that moves chunks to equalize data distribution.

## 4. Sharding modes

- Range sharding: contiguous shard key ranges are assigned to shards. Good for range queries, can create hot chunks with monotonically increasing keys.
- Hashed sharding: values are hashed before distribution. Good for even write distribution, bad for range queries and sort operations.

### Example sharding commands

```js
sh.enableSharding('sales');
sh.shardCollection('sales.orders', { orderDate: 1 });
```

```js
sh.shardCollection('sales.orders', { orderId: 'hashed' });
```

## 5. Why shard keys matter

- The shard key controls physical placement of documents.
- A poor shard key can lead to uneven chunk distribution, jumbo chunks, high balancer activity, and poor query routing.
- The shard key also affects which queries can be targeted to one shard.

## 6. Sharding overhead and tradeoffs

- Every query may require an extra `mongos` hop, a possible `SHARD_MERGE`, and a `SHARDING_FILTER`.
- Writes incur additional metadata lookups and potential chunk splits or migrations.
- Chunk migration moves data in the background, increasing network and disk usage.
- Zones and tag-aware sharding add valuable control at the cost of additional planning.

## 7. Answering style

- Emphasize that sharding is for write scaling and that it adds overhead.
- Use the phrase “last resort” when describing when to shard.
- Explain the difference between replicas and shards clearly: replicas provide redundancy; shards provide scale.
- Provide commands for `sh.enableSharding`, `sh.shardCollection`, and `sh.status()` as canonical examples.
- Mention that all previous single-node tuning advice still applies under sharding.
