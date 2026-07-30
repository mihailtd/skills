---
name: database-mongodb-sharded-query-tuning
description: Guides the agent on tuning queries for MongoDB sharded clusters, including shard targeting, explain plans, sorts, and aggregation behavior.
---

# MongoDB Sharded Query Tuning

You are an expert on MongoDB sharded query tuning. When asked about sharded query performance, explain shard targeting, execution plans, index support, and aggregation behavior.

## 1. Sharded explain plans

- Use `explain('executionStats')` to see how a query is executed across shards.
- Important stages:
  - `SINGLE_SHARD`: query resolved on one shard
  - `SHARD_MERGE`: results merged from multiple shards
  - `SHARD_MERGE_SORT`: sorted merge across shards
  - `SHARDING_FILTER`: filter applied after shard selection or when scanning
- A `SHARD_MERGE` is not inherently bad; it is normal when the shard key is absent.
- The cost comes from work done on every shard and the merge operation.

## 2. Targeted queries and shard key lookups

- Queries containing the shard key are more likely to target a single shard.
- When the shard key is present with a supporting index, the query can avoid broadcasting to all shards.
- For compound shard keys, include the prefix fields in the query filter.

### Example targeted query

```js
var exp = db.customersShardName.explain('executionStats').find({ LastName: 'SMITH', FirstName: 'MARY' });
printjson(exp.executionStats);
```

- If a compound index exists on `{ LastName: 1, FirstName: 1 }`, the query can use a single-shard plan.

## 3. Non-shard-key queries

- Queries without the shard key are sent to every shard.
- Each shard executes the query locally, then results are merged.
- Optimize these queries by ensuring each shard has good indexes and compact documents.

### Example non-shard-key query

```js
var exp = db.customersSCountry.explain('executionStats').find({ 'views.filmId': 637 });
```

- A `SHARD_MERGE` result is expected.
- The key is to minimize per-shard work with indexes, projections, and efficient document design.

## 4. Range queries and shard mode

- Range queries on range-sharded keys can often target a subset of shards.
- Range queries on hashed shard keys usually broadcast to all shards because hash breaks ordering.

### Range targetable example

```js
var startDate = ISODate('2018-01-01T00:00:00.000Z');
var exp = db.ordersSOrderDate.explain('executionStats').find({ orderDate: { $gt: startDate } });
```

- This may execute on a single shard if the range lies entirely inside one chunk.

### Hashed shard key example

```js
var exp = db.ordersHOrderDate.explain('executionStats').find({ orderDate: { $gt: startDate } });
```

- This query usually requires scanning all shards.

## 5. Sorting in sharded queries

- Sorting across shards triggers `SHARD_MERGE_SORT` on `mongos`.
- If each shard can sort using an index, shard-local sorting is cheaper.
- If no index supports the sort, each shard performs a blocking sort, worsening the merge cost.

### Example sort behavior

- With `orderDate_1` support on each shard, the query can sort locally before merge.
- Without support, each shard incurs a `SORT` stage.

## 6. Aggregation behavior

- Aggregation plans include `mergeType`, `splitPipeline`, and `mergerPart`.
- MongoDB pushes eligible work to shards and merges results on `mongos` or a shard.
- `$match`, `$project`, `$group`, and `$unwind` can be shard-aware if the pipeline is splittable.

### Sharded aggregation example

```js
db.customersSCountry.aggregate([
  { $unwind: '$views' },
  { $group: { _id: { views_title: '$views.title' }, count: { $sum: 1 } } }
]);
```

- The explain output shows shardsPart work on each shard and mergerPart on the router.

## 7. $lookup limitations on sharded collections

- A `$lookup` stage cannot reference a sharded collection in the `from` field when the pipeline is distributed.
- The initiating collection may be sharded, but the lookup collection must be unsharded.
- If the lookup collection is sharded, the operation is performed on the primary shard of the initiating collection and can become a bottleneck.

## 8. Index recommendations

- Create indexes that combine the shard key with additional filter or sort fields.
- Example:

```js
db.customers.createIndex({ LastName: 1, FirstName: 1 });
```

- For queries that use the shard key plus other predicates, a compound index can reduce fetches and allow single-shard execution.

## 9. Explain plan analysis tips

- Compare `nReturned`, `keysExamined`, and `docsExamined` across shards.
- A `SHARDING_FILTER` with many docs returned may indicate an inefficient partial match.
- Use `executionTimeMillis` from each shard to spot imbalance.
- For sharded aggregates, inspect `mergeType` to see whether `mongos` or a shard is merging.

## 10. Answering style

- Advise always to include the shard key in queries when possible.
- Explain that `SHARD_MERGE` is normal but should be supported by efficient per-shard work.
- Use examples that compare `SINGLE_SHARD` and `SHARD_MERGE` plans.
- Mention the effect of hashed shard keys on range queries and sorts.
- Call out `$lookup` sharded limitations explicitly.
