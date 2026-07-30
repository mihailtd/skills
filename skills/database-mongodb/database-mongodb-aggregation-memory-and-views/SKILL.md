---
name: database-mongodb-aggregation-memory-and-views
description: Guides the agent on MongoDB aggregation memory limits, disk sorts, views, and materialized view patterns.
---

# MongoDB Aggregation Memory and Views

You are an expert on MongoDB aggregation memory management and view performance. When asked about memory limits, disk sorts, views, or materialized views, explain the constraints, tuning options, and performance trade-offs.

## 1. Aggregation memory limits

- Aggregation pipelines are subject to a 100MB memory limit by default.
- If a pipeline exceeds that limit, MongoDB returns an error unless `allowDiskUse: true` is enabled.
- The 16MB BSON document size limit still applies to any intermediate or final document produced by the pipeline.

### Example

```js
db.customers.aggregate(pipeline, { allowDiskUse: true });
```

### Tip

- Use `allowDiskUse: true` when large sorts, groupings, or joins are expected.
- Do not rely on disk spill as the first solution; optimize the pipeline first.

## 2. Sorts in aggregation

- A sort can be optimized by an index if it occurs early enough in the pipeline.
- If the sort is after transformation stages like `$addFields`, `$project`, or `$group`, the index is usually unavailable.

### Example: indexed sort

```js
db.baseCollection.aggregate([
  { $sort: { d: 1 } },
  { $addFields: { x: 0 } }
], { allowDiskUse: true });
```

### Example: disk sort

```js
db.baseCollection.aggregate([
  { $addFields: { x: 0 } },
  { $sort: { d: 1 } }
], { allowDiskUse: true });
```

- The latter may require a disk sort and be much slower.
- Move sort before transformations when possible.

## 3. Disk sort behavior

- Disk sorts are slower but allow the query to complete when memory is insufficient.
- Use `allowDiskUse: true` to permit disk-based sorting.
- The performance can jump abruptly once the sort spills to disk.

### Tip

- If a sort consistently spills to disk, try increasing memory limits or restructuring the pipeline.
- Avoid disk sorts for latency-sensitive queries.

## 4. Increasing sort memory limits

- MongoDB supports internal parameters such as `internalQueryMaxBlockingSortMemoryUsageBytes`.
- Increasing these values should be done with care.

### Example

```js
db.getSiblingDB('admin').runCommand({
  setParameter: 1,
  internalQueryMaxBlockingSortMemoryUsageBytes: 1048576000
});
```

- This raises the blocking sort limit to 1GB.
- Monitor overall server memory to avoid exhaustion.

## 5. Document size considerations

- Aggregation output documents must still fit under the 16MB BSON limit.
- If a document temporarily exceeds 16MB but is trimmed before the end of the pipeline, no error occurs.
- If `$lookup` returns a massive array that is later `$unwind`ed, MongoDB may avoid the size error.

### Tip

- Avoid growing arrays or embedding large subdocuments in intermediate stages.
- Trim unneeded fields with `$project` before group/lookup stages.

## 6. Views and performance

- MongoDB views are read-only aggregations over base collections.
- A view does not store data separately; it replays the underlying pipeline on each query.
- Views do not inherently improve performance; they simplify query syntax.

### Example

```js
db.createView('productTotals', 'lineitems', [
  { $group: { _id: '$prodId', total: { $sum: '$itemCount' } } }
]);
```

- Querying `productTotals` still executes the underlying aggregation.
- Views may suppress index usage compared to querying the source collection directly.

## 7. Materialized views

- Materialized views store the results of an aggregation in a collection.
- Use `$merge` or `$out` to build materialized views.
- They trade freshness for query speed.

### Example using `$merge`

```js
db.lineitems.aggregate([
  { $group: { _id: '$prodId', orderCount: { $sum: '$itemCount' } } },
  { $merge: { into: 'salesByCityMV' } }
]);
```

### Tip

- Refresh materialized views on a schedule or via change streams.
- Do not refresh more often than necessary.
- Use materialized views for expensive aggregations that are queried frequently.

## 8. View limits and index visibility

- Queries against views may not use indexes on the source collection as effectively.
- If a query against a view is slow, compare performance against the underlying aggregation directly.
- In some cases, querying the base collection with the pipeline manually is faster.

## 9. Memory-bound stage notes

- `$addToSet` and `$push` accumulators do not spill to disk.
- If these stages exceed memory, you must change the pipeline or pre-aggregate the data.
- `$group` and `$sort` can spill to disk if `allowDiskUse` is enabled.

## 10. Answering style

- Explain that memory limits and disk sorts are the main aggregation scaling concerns.
- Mention 100MB default memory limit and 16MB document limit explicitly.
- Use `allowDiskUse`, `$merge`, and `$out` in examples.
- Distinguish views from materialized views clearly.
- Advise careful use of internal memory parameters.
