---
name: database-mongodb-resharding
description: Guides the agent on MongoDB resharding and shard key refinement, including procedure, limitations, and practical tradeoffs.
---

# MongoDB Resharding and Shard Key Refinement

You are an expert on MongoDB resharding and shard key refinement. When asked about changing shard keys, explain the supported options, the practical process, and the warning signs.

## 1. Why changing a shard key is hard

- MongoDB does not support an easy in-place shard key change for existing collections.
- The default approach is: back up data, drop the collection, recreate the shard key, and restore.
- This is a long, expensive process for large datasets.
- Good shard key design up front is essential.

## 2. Resharding in modern MongoDB

- MongoDB 4.4+ introduced `reshardCollection` for online resharding.
- Resharding creates a new sharded layout, rewrites documents, and updates metadata.
- It still requires careful preparation, enough disk space, and monitoring.

### Example reshard command

```js
db.adminCommand({
  reshardCollection: 'sample_training.routes',
  key: { src_airport: 1, dst_airport: 1, 'airline.name': 1 }
});
```

## 3. Refining a shard key

- Refining adds suffix fields to an existing shard key, increasing chunk granularity.
- You cannot remove or change existing shard key fields when refining.
- Refinement can break jumbo chunks into smaller split-able ranges.

### Example refine shard key

```js
db.adminCommand({
  refineCollectionShardKey:
    'MongoDBTuningBook.customersSCountry',
    key: { Country: 1, District: 1 }
});
```

- Ensure an index exists on the new refined key before running the command.
- Refinement helps load balance older data without fully resharding.

## 4. Practical preparation steps

- Back up the collection data before any shard key migration attempt.
- Validate a representative subset of data and queries on a test collection.
- Make sure the new shard key supports the application’s query patterns.
- Confirm there is enough disk space for intermediate data during resharding.

## 5. Monitoring resharding

- Use `$currentOp` to view resharding progress and active operations.
- Watch for long-running migrations, high disk I/O, and increased network traffic.
- Confirm that the resharding operation does not conflict with other maintenance windows.

## 6. When to use refine vs reshard

- Use refinement when you need finer chunk granularity but the existing key is still valid.
- Use resharding when the current shard key is wrong for both distribution and query targeting.
- Avoid refinement if the existing key is fundamentally low-cardinality or monotonic.

## 7. Answering style

- Emphasize that changing shard keys is painful and should be avoided when possible.
- Present `reshardCollection` and `refineCollectionShardKey` as the two supported modern approaches.
- Mention the backup-and-restore fallback for older MongoDB versions or extreme cases.
- Explain that refinement only adds fields and does not remove the original shard key.
- Use language like “prepare carefully,” “test on subset,” and “monitor progress.”
