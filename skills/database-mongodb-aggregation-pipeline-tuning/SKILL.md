---
name: database-mongodb-aggregation-pipeline-tuning
description: Guides the agent on tuning MongoDB aggregation pipelines, including explain output, stage ordering, pipeline optimization, and early filtering techniques.
---

# MongoDB Aggregation Pipeline Tuning

You are an expert on MongoDB aggregation tuning. When asked about aggregation pipeline performance, explain how to interpret pipeline explain plans, how to reduce intermediate data, and how to design stage order for efficient execution.

## 1. Aggregation explain basics

- Aggregation `explain()` output differs from `find()` explain output.
- The top-level `stages` array contains each aggregation stage and the query planner information for the cursor source.
- The initial data access is usually represented by a `$cursor` stage inside `stages[0]`.
- Use `explain('executionStats')` to understand actual work done.

### Example

```js
db.customers.explain('executionStats').aggregate([
  { $match: { Country: 'Japan', LastName: 'Smith' } },
  { $project: { _id: 0, FirstName: 1, LastName: 1 } },
  { $sort: { FirstName: -1 } },
  { $limit: 3 }
]);
```

- In the explain output, the engine may show a `$cursor` stage followed by a `$sort` stage.
- The `$cursor` stage contains `queryPlanner.winningPlan` describing index usage.
- Subsequent pipeline stages are shown as separate stage objects.

## 2. Filter early, reduce data quickly

- The earlier you remove documents from the pipeline, the less work subsequent stages perform.
- Push `$match` stages as early as possible, ideally before `$lookup`, `$group`, `$unwind`, `$project`, or `$addFields`.
- MongoDB can sometimes reorder `$match` stages automatically, but you should still write pipelines with early filtering.

### Example

Inefficient:

```js
db.lineitems.aggregate([
  { $lookup: { from: 'orders', localField: 'orderId', foreignField: '_id', as: 'orders' } },
  { $sort: { itemCount: -1 } },
  { $limit: 5 }
]);
```

Better:

```js
db.lineitems.aggregate([
  { $sort: { itemCount: -1 } },
  { $limit: 5 },
  { $lookup: { from: 'orders', localField: 'orderId', foreignField: '_id', as: 'orders' } }
]);
```

- This reduces the amount of data entering `$lookup`.
- If the pipeline includes a `$limit`, place it before expensive joins and group stages when safe.

## 3. Pipeline stage order

- The canonical optimization pattern is: filter → project → aggregate → sort → limit.
- Stages that reduce document count should come before stages that transform or expand documents.
- Avoid placing `$project` before `$match` unless the projection is required to produce the filter fields.

### Good ordering pattern

- `$match`
- `$sort`
- `$limit`
- `$group`
- `$project`
- `$lookup`
- `$unwind`

### When to move `$sort` and `$limit`

- If `$sort` has an index, move it as close to the start as possible.
- If `$limit` is known early, push it up to reduce work in later stages.

## 4. Automatic pipeline optimizations

- MongoDB can merge adjacent `$match`, `$limit`, and `$skip` stages.
- `$lookup` followed immediately by `$unwind` may be merged internally.
- `$sort` followed by `$limit` can be optimized to maintain only the required top N documents.
- MongoDB may insert hidden projections to remove unused fields.

### Example of automatic merge

```js
db.listingsAndReviews.explain('executionStats').aggregate([
  { $match: { property_type: 'House' } },
  { $match: { bedrooms: 3 } },
  { $limit: 100 },
  { $limit: 5 },
  { $skip: 3 },
  { $skip: 2 }
]);
```

- The engine can collapse this into a single `$match`, one `LIMIT`, and one `SKIP`.
- `mongoTuning.aggregationExecutionStats()` summarizes the actual stage timing.

## 5. Reading aggregation explain output

- Start with `stages[0].$cursor.queryPlanner.winningPlan` for the initial scan.
- Look for `IXSCAN` or `COLLSCAN` in the cursor plan.
- Check whether `$group`, `$sort`, `$lookup`, or `$unwind` appear later.
- Compare `nReturned`, `docsExamined`, and `keysExamined` to understand selectivity.

### Example

```js
1 IXSCAN (Country_1_LastName_1 ms:0 keys:21368 nReturned:21368)
2 FETCH (ms:13 docsExamined:21368 nReturned:21368)
3 PROJECTION_SIMPLE (ms:15 nReturned:21368)
4 $GROUP (ms:70 returned:31)
5 $SORT (ms:70 returned:10)
```

- If the pipeline plan shows `PROJECTION_SIMPLE` before `$GROUP`, MongoDB is trimming unused fields.
- If the plan shows `IXSCAN` followed by `FETCH`, the initial stage is using an index.

## 6. Optimizing expensive stages

- `$group` and `$sort` are expensive because they materialize intermediate state.
- Use `$match` before `$group` whenever possible.
- Use an index to support `$sort` if the sort occurs early in the pipeline.
- Avoid `$project` or `$addFields` before a sort whenever possible.

### Example

Inefficient:

```js
db.baseCollection.aggregate([
  { $addFields: { x: 0 } },
  { $sort: { d: 1 } }
], { allowDiskUse: true });
```

Better:

```js
db.baseCollection.aggregate([
  { $sort: { d: 1 } },
  { $addFields: { x: 0 } }
], { allowDiskUse: true });
```

- The better pipeline can use an index for sorting, while the former may require a disk sort.

## 7. `$lookup` and join placement

- Place filtering before `$lookup` to reduce the number of joined documents.
- If you can trim the input set before the join, do so.
- A `$lookup` is executed once for each input document.

### Example

Use this pattern when filtering on the local collection:

```js
db.lineitems.aggregate([
  { $match: { status: 'shipped' } },
  { $lookup: { from: 'products', localField: 'prodId', foreignField: '_id', as: 'product' } }
]);
```

- If the filter is on the foreign collection, consider whether the join order can be reversed.

## 8. `allowDiskUse` and memory control

- Aggregation pipelines have a 100MB memory limit by default.
- Use `allowDiskUse: true` to allow operations to spill to disk.
- `$group` and `$sort` can spill to disk; `$addToSet` and `$push` do not spill.

### Example

```js
db.customers.aggregate(pipeline, { allowDiskUse: true });
```

### Tip

- Use `allowDiskUse` when large intermediate data is unavoidable.
- Prefer pipeline restructuring before relying on disk spill.

## 9. Tuning `graphLookup`

- `$graphLookup` traverses recursive relationships and can explode rapidly with `maxDepth`.
- Index `connectToField` to make traversal efficient.
- Reduce the seed set to only the required starting nodes.

### Example

```js
db.socialGraph.aggregate([
  { $match: { person: 1476767 } },
  { $graphLookup: {
      from: 'socialGraph',
      startWith: [1476767],
      connectFromField: 'friends',
      connectToField: 'person',
      maxDepth: 2,
      depthField: 'Depth',
      as: 'GraphOutput'
    } },
  { $unwind: '$GraphOutput' }
], { allowDiskUse: true });
```

- A `connectToField` index is essential for scale.
- Join expansion into large graphs can be orders of magnitude more expensive without indexing.

## 10. Answering style

- Explain aggregation tuning as a process of reducing pipeline input size and minimizing intermediate data.
- Use concrete stage-order guidance: filter early, sort/limit before joins when safe.
- Mention automatic pipeline reordering and hidden projections.
- Provide examples showing both bad and good pipeline order.
- Call out when `allowDiskUse` is needed and when it should be a fallback.
