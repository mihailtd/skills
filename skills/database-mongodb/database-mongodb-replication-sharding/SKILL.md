---
name: database-mongodb-replication-sharding
description: Guides the agent on MongoDB replica sets, oplog replication, change streams, Atlas sharded clusters, shard-key selection, and high-availability operations.
---

# MongoDB Replication and Sharding

You are an expert on MongoDB high availability, replication, and sharding. When asked about replica sets, oplogs, change streams, or Atlas sharded clusters, explain the core concepts, operational commands, and architectural tradeoffs.

## 1. Replication vs sharding

- Replication copies the same data set across multiple nodes to provide high availability and read scaling.
- Sharding partitions the data set across multiple shards for horizontal scale.
- Each shard is itself a replica set.
- In Atlas, replication is automatic; sharding is optional and requires an M30+ tier.

## 2. Replica set basics

- A replica set is a group of `mongod` instances with identical data.
- One member is `PRIMARY`; others are `SECONDARY`.
- Replica set states include:
  - `PRIMARY` (1)
  - `SECONDARY` (2)
  - `ARBITER` (6)
  - `STARTUP`, `STARTUP2`, `RECOVERING`, `ROLLBACK`, `DOWN`, `UNKNOWN`, `REMOVED`
- Use `rs.status()` or `db.adminCommand({ replSetGetStatus: 1 })` to inspect status.

### Example status command

```js
db.adminCommand('replSetGetStatus').members.map(m => ({
  _id: m._id,
  name: m.name,
  state: m.state,
  stateStr: m.stateStr
}));
```

### Replica set config

- Use `rs.conf()` or `db.adminCommand({ replSetGetConfig: 1 })` to view config.
- Important fields:
  - `arbiterOnly`
  - `hidden`
  - `priority`
  - `secondaryDelaySecs`
  - `votes`
- Hidden members are useful for analytics or backup nodes.
- Priority 0 nodes cannot become primary; they can be nonvoting and still replicate.

## 3. Elections and failover

- MongoDB uses PV1 (Raft-like) for elections.
- Elections occur when:
  - a primary is unavailable
  - nodes are added/removed
  - heartbeats fail for > 10 seconds
- The most up-to-date secondary has the best chance of election.
- Applications should handle automatic failover and retry logic.

### Election tuning tip

- `settings.electionTimeoutMillis` controls election timeout.
- Do not make it too small or your cluster may flap.

## 4. Oplog internals

- The oplog is a capped collection (`local.oplog.rs`) containing the write history.
- Primary writes are logged here, then streamed to secondaries.
- Every oplog entry is idempotent.
- Chained replication allows secondaries to replicate from other secondaries.
- Use `db.getReplicationInfo()` to inspect oplog size and time window.

### Sample oplog document

```js
{
  op: 'i',
  ns: 'sample_mflix.sessions',
  ui: new UUID('...'),
  o: { _id: ObjectId('...'), user_id: '12345', jwt: 'token123456' },
  ts: Timestamp({ t: 1720567182, i: 1 }),
  t: Long('39'),
  v: Long('2'),
  wall: ISODate('2024-03-15T10:20:30.123Z')
}
```

### Oplog sizing

- Default size is 5% of disk for WiredTiger, min 990 MB, max 50 GB.
- For in-memory storage, default is 5% of RAM, min 50 MB.
- Use `replSetResizeOplog` to resize after initialization.
- Keep the oplog window long enough to support delayed members and initial sync.

### Replication lag check

- `db.printSecondaryReplicationInfo()` for shell diagnostics.
- For automation, use `rs.status()` and parse `optimeDate`.

## 5. Change streams

- Change streams expose real-time data changes on collections, databases, or deployments.
- They require replica sets or sharded clusters on WiredTiger with PV1.
- Events include `insert`, `update`, `replace`, `delete`, and `invalidate`.
- The event `_id` is the resume token.

### Simple mongosh watcher

```js
const watchCursor = db.getSiblingDB('sample_mflix').watch();
while (!watchCursor.isClosed()) {
  let next = watchCursor.tryNext();
  while (next !== null) {
    printjson(next);
    next = watchCursor.tryNext();
  }
}
```

### Node.js change stream example

```js
const { MongoClient } = require('mongodb');
const client = new MongoClient(uri, { serverApi: '1' });

async function watchSessions() {
  await client.connect();
  const changeStream = client.db('sample_mflix').collection('sessions').watch();
  changeStream.on('change', change => {
    console.log('Change event:', change);
  });
}
```

### Filtering change stream output

```js
const pipeline = [
  { $match: { 'fullDocument.user_id': '12345' } },
  { $addFields: { filteredAt: new Date() } }
];
const changeStream = sessionsCollection.watch(pipeline);
```

### Resume tokens

- Never modify `_id` in change stream pipelines.
- Use `resumeAfter` or `startAfter` to resume after disconnects.
- Persist the resume token if you require reliable processing.

## 6. Sharded cluster fundamentals

- A sharded cluster consists of:
  - shards (replica sets)
  - `mongos` routers
  - config servers or embedded config servers in MongoDB 8.0+
- `mongos` routes queries to shards and merges results.
- Use `show dbs` after connecting to `mongos`.

## 7. Atlas sharded cluster commands

### Create a sharded cluster with Atlas CLI

```bash
atlas clusters create "MongoDB-in-Action-Sharded" \
  --provider GCP --region CENTRAL_US --tier M30 --type SHARDED --shards 2 --mdbVersion 8.0
```

### Load sample data

```bash
atlas clusters sampleData load "MongoDB-in-Action-Sharded"
```

### Show connection string

```bash
atlas clusters connectionStrings describe "MongoDB-in-Action-Sharded"
```
```

## 8. Shard key selection

- Shard keys determine chunk distribution.
- Good shard keys are high-cardinality, evenly distributed, and match query patterns.
- Avoid monotonically increasing keys for range sharding.
- Use hashed shard keys to avoid hotspots, but not for range queries.
- Shard key index rules:
  - must start with the shard key
  - cannot be descending
  - cannot be partial, geospatial, multikey, text, or wildcard

## 9. Shard key analyzer

- Use `analyzeShardKey()` in MongoDB 7.0+ to evaluate candidate keys.
- Enable query sampling first with `configureQueryAnalyzer()`.

### Example pipeline

```js
db.routes.createIndex({ src_airport: 1, dst_airport: 1, 'airline.name': 1 });
db.routes.configureQueryAnalyzer({ mode: 'full', samplesPerSecond: 1 });
const result = db.routes.analyzeShardKey({ src_airport: 1, dst_airport: 1, 'airline.name': 1 });
printjson(result);
```

## 10. Shard administration commands

### Enable sharding and shard a collection

```js
sh.enableSharding('sample_training');
sh.shardCollection('sample_training.routes', { src_airport: 1, dst_airport: 1, 'airline.name': 1 });
```

### Check sharding status

```js
sh.status();
db.getSiblingDB('config').chunks.aggregate([
  { $group: { _id: '$shard', totalSize: { $sum: '$size' }, count: { $sum: 1 } } }
]);
```

### Get shard distribution

```js
db.routes.getShardDistribution();
```

## 11. Balancing and chunk management

- Chunks are ranges of shard-key values.
- Default chunk size is 128 MB in MongoDB 5.2+.
- The balancer moves chunks for even distribution.
- Balancer windows reduce disruption.

### Balancer control

```js
sh.startBalancer();
sh.stopBalancer();
sh.isBalancerRunning();
sh.getBalancerState();
```

### Set balancing window

```js
use config;
db.settings.updateOne(
  { _id: 'balancer' },
  { $set: { activeWindow: { start: '02:00', stop: '06:00' } } },
  { upsert: true }
);
```

## 12. Resharding and unsharding

- Resharding changes the shard key for an existing collection.
- It requires extra disk space and careful prep.
- Use `reshardCollection` with a new key.
- Monitor with `$currentOp`.

### Reshard command

```js
db.adminCommand({
  reshardCollection: 'sample_training.routes',
  key: { src_airport: 1, dst_airport: 1, 'airline.name': 1 }
});
```

### Unshard command

```js
db.adminCommand({
  unshardCollection: 'sample_training.routes',
  toShard: 'shard0000'
});
```

## 13. Advanced MongoDB 8.0 sharded features

- Embedded config servers simplify topology for small clusters.
- Move unsharded collections with `moveCollection`.
- AutoMerger merges eligible contiguous chunks.
- Sharding metadata is stored in config servers or config shard.

### Move unsharded collection

```js
db.adminCommand({ moveCollection: 'sample_training.zips', toShard: 'shard1' });
```

### AutoMerger control

```js
sh.startAutoMerger();
sh.stopAutoMerger();
```

## 14. Consistency and availability

- Write Concern controls write acknowledgement.
- Read Concern controls consistency of reads.
- Read Preference controls which members serve reads.

### Write Concern examples

```js
db.routes.insertOne(
  { src_airport: 'MUC' },
  { writeConcern: { w: 'majority', j: true, wtimeout: 5000 } }
);
```

### Read Concern examples

```js
db.routes.find({ src_airport: 'MUC' }).readConcern('majority');
```

### Read Preference examples

```js
db.routes.find({ src_airport: 'MUC' }).readPref('secondaryPreferred');
```

## 15. Tips and tricks

- Use `replSetGetStatus` for health and replication lag.
- Use `rs.printReplicationInfo()` for manual debugging and `db.getReplicationInfo()` for scripts.
- Hide indexes before dropping them in production.
- Use `analyzeShardKey()` before sharding.
- Avoid shard keys with low cardinality or monotonic ranges.
- Prefer automatic balancer migration unless manual splits are necessary.
- When resharding, keep the application aware of both old and new shard keys until complete.
- Use `secondaryPreferred` or `nearest` read preference for analytics workloads.

## 16. Answering style

- Distinguish replication from sharding clearly.
- Use `rs.status()`, `replSetGetConfig`, `sh.status()`, `analyzeShardKey()`, and `reshardCollection()` as canonical examples.
- Mention Atlas defaults for replication and the M30+ requirement for sharding.
- Explain shard key tradeoffs, oplog windows, and change stream resume tokens.
