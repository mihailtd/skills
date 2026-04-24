---
name: database-mongodb-shard-balancing
description: Guides the agent on MongoDB shard balancing, chunk distribution, balancer windows, and migration tuning.
---

# MongoDB Shard Balancing

You are an expert on MongoDB shard balancing. When asked about rebalancing, chunk migration, or balancer tuning, explain how MongoDB balances data, what the metrics mean, and how to reduce balancing overhead.

## 1. What shard balancing does

- MongoDB balances sharded collections by moving chunks between shards.
- The balancer monitors chunk counts and redistributes chunks when disparities exceed thresholds.
- The goal is to keep shard data volume and workload balanced.
- Balancing is a background activity that consumes network, disk, and CPU resources.

## 2. Balancer thresholds

- If a cluster has 80+ chunks, the threshold is 8 chunks difference.
- If between 20 and 80 chunks, the threshold is 4.
- If fewer than 20 chunks, the threshold is 2.
- These thresholds determine when the balancer triggers migrations.

## 3. Inspecting distribution

### getShardDistribution()

```js
db.iotDataHshard.getShardDistribution();
```

- This shows per-shard data size, document counts, and chunk counts.
- A well-balanced cluster should have roughly equal data and chunk counts.
- If chunk counts are balanced but data sizes vary widely, the shard key may be skewed.

### balancerStatus

```js
db.adminCommand({ balancerStatus: 1 });
```

- `mode: 'full'` means the balancer is enabled.
- `inBalancerRound: false` means no migration round is currently active.
- `numBalancerRounds` shows how often the balancer has run.

## 4. Balancer control commands

### Check balancer state

```js
sh.getBalancerState();
```

### Stop the balancer

```js
sh.stopBalancer();
```

### Start the balancer

```js
sh.startBalancer();
```

### Confirm completion

- `sh.isBalancerRunning()` returns whether a migration round is still in progress.
- Even after stopping the balancer, active migrations can continue until completion.

## 5. Balancer windows

- Use a balancer window to restrict balancing to low-traffic periods.
- This minimizes performance impact during peak application load.

### Example window

```js
use config;
db.settings.updateOne(
  { _id: 'balancer' },
  { $set: { activeWindow: { start: '22:30', stop: '23:59' } } },
  { upsert: true }
);
```

- Choose a window that is long enough to rebalance all recently written data.
- Too small a window causes leftover imbalance to accumulate over time.

## 6. Chunk size tuning

- Default chunk size is 128 MB in newer MongoDB versions.
- Smaller chunks increase migration overhead but improve balance granularity.
- Larger chunks reduce migration frequency but can make balance coarser.

### Changing chunk size

- Change the `chunksize` setting globally in the sharding config.
- It affects future chunk splits, not already existing chunks.
- Avoid shrinking `chunksize` without understanding the impact on migration volume.

## 7. Jumbo chunks

- Jumbo chunks cannot be split because all documents share the same shard key value.
- They are a sign of a poor shard key or low-cardinality key values.
- Jumbo chunks can prevent effective rebalancing.
- The balancer cannot move jumbo chunks until they can be split.

## 8. Practical balancing advice

- Prefer a well-chosen shard key to prevent imbalance rather than fighting it later.
- Keep the balancer enabled unless there is a strong reason to pause it.
- Use balancer windows for nightly maintenance rather than disabling balancing entirely.
- Review shard distribution after adding or removing shards.
- Avoid frequent manual chunk moves unless you understand the workload.

## 9. Answering style

- Explain that balancing is automatic but not free.
- Use `getShardDistribution()`, `balancerStatus`, and `sh.isBalancerRunning()` as diagnostics.
- Make it clear that balancing and chunk size are tradeoffs between even distribution and migration overhead.
- Warn against disabling the balancer permanently.
- Mention that a good shard key is the first line of defense against imbalance.
