---
name: database-mongodb-aggregation-pipeline-stages
description: Guides the agent on MongoDB aggregation pipeline stages, stage ordering, stage-specific use cases, and how to combine stages for robust pipelines.
---

# MongoDB Aggregation Pipeline Stages

You are an expert on MongoDB aggregation stages. When asked about stage selection or pipeline design, explain the purpose, input/output shape, and best placement for each stage.

## 1. Core stages and when to use them

### `$match`

- Filters documents.
- Best placed early.
- Use indexed fields to maximize performance.

Example:

```js
{ $match: { airplane: 'CR2', stops: 0 } }
```

### `$group`

- Aggregates documents into groups.
- Use accumulator operators like `$sum`, `$avg`, `$min`, `$max`, `$push`, `$addToSet`, and `$first`.

Example:

```js
{ $group: { _id: '$src_airport', totalRoutes: { $sum: 1 } } }
```

### `$project`, `$set`, `$unset`

- `$project` reshapes the entire document.
- `$set` adds or modifies fields while preserving others.
- `$unset` removes fields.

Example:

```js
{ $set: { isDirect: { $eq: ['$stops', 0] }, codeshare: '$$REMOVE' } }
```

### `$sort`

- Orders documents.
- Use indexes on the sort fields when possible.
- Put costly sorts after `$match` if the filtered subset is much smaller.

Example:

```js
{ $sort: { totalRoutes: -1, src_airport: 1 } }
```

### `$skip` and `$limit`

- `$skip` is used for pagination or skipping a fixed number of documents.
- `$limit` stops the pipeline after the first N documents.
- Combine `$sort` + `$skip` + `$limit` for paging.

Example:

```js
[{ $sort: { date: -1 } }, { $skip: 20 }, { $limit: 10 }]
```

### `$unwind`

- Flattens arrays into separate documents.
- Use `preserveNullAndEmptyArrays: true` when you want to keep documents missing the array.

Example:

```js
{ $unwind: { path: '$accounts', preserveNullAndEmptyArrays: true } }
```

### `$lookup`

- Performs a left outer join with another collection.
- Use `localField` and `foreignField` for simple joins.
- Use `pipeline` and `let` for more complex lookups.

Example:

```js
{ $lookup: { from: 'customers', localField: 'account_id', foreignField: 'accounts', as: 'customer_details' } }
```

### `$unionWith`

- Combines results from another collection or pipeline.
- Useful for union-style queries.

Example:

```js
{ $unionWith: { coll: 'archived_routes', pipeline: [{ $match: { active: false } }] } }
```

## 2. Advanced stage groups

### $setWindowFields

- Adds computed windowed values such as running totals and moving averages.
- Use after sorting and grouping by partition keys.

Example:

```js
{
  $setWindowFields: {
    partitionBy: '$src_airport',
    sortBy: { timestamp: 1 },
    output: { cumulativeRoutes: { $sum: 1, window: { documents: ['unbounded', 'current'] } } }
  }
}
```

### `$facet`

- Runs multiple pipelines in parallel on the same input.
- Use for dashboards or multi-result reports.

Example:

```js
{
  $facet: {
    routeCounts: [{ $group: { _id: '$src_airport', count: { $sum: 1 } } }],
    topAirlines: [{ $group: { _id: '$airline.name', count: { $sum: 1 } } }, { $sort: { count: -1 } }, { $limit: 5 }]
  }
}
```

### `$bucket` and `$bucketAuto`

- Use `$bucket` to categorize documents into explicit ranges.
- Use `$bucketAuto` to create evenly distributed buckets automatically.

Example:

```js
{
  $bucketAuto: {
    groupBy: '$passenger_count',
    buckets: 5,
    output: { count: { $sum: 1 } }
  }
}
```

## 3. Best placement patterns

- `$match` early to reduce volume.
- `$project` late if the document shape changes strongly.
- `$sort` after `$match`, before `$limit`.
- `$unwind` after filtering arrays when appropriate.
- `$lookup` after `$match` if you can avoid joining unnecessary documents.
- `$group` after `$unwind` when you aggregate array elements.

## 4. Performance tips

- Keep pipeline stages small and focused.
- Avoid bringing back large arrays unless required.
- Use `$unset` to remove temporary fields used only for intermediate calculations.
- Check stage order with `explain('executionStats')`.
- If a stage is expensive, consider rewriting the pipeline or precomputing fields.

## 5. Stage usage cheatsheet

| Stage | Use case | Tip |
|---|---|---|
| `$match` | Filter input docs | Put first whenever possible |
| `$group` | Summarize data | Keep grouping keys small |
| `$project` | Reshape output | Use last if removing fields |
| `$set` | Add derived fields | Use with `$$REMOVE` to drop fields |
| `$unset` | Remove fields | Cleaner than `$project` for small deletions |
| `$sort` | Order docs | Use indexed fields |
| `$limit` | Cap results | Use after `$sort` |
| `$skip` | Pagination | Avoid deep skips on large data |
| `$unwind` | Flatten arrays | Filter arrays first |
| `$lookup` | Join collections | Index foreign field |
| `$unionWith` | Merge pipelines | Good for archiving or multitenant data |
| `$out` | Persist results | Replaces target collection |
| `$merge` | Upsert results | Useful for incremental refresh |

## 6. Answering style

- Define the stage, the input shape, and the output shape.
- Provide a short code snippet for each stage.
- Mention stage-specific costs, such as `$sort` requiring memory and indices.
- For nontrivial pipelines, recommend `explain()` and stage reordering.
- Use `routes` / `customers` / `transactions` examples when possible.
