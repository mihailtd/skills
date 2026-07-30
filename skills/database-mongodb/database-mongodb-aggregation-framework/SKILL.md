---
name: database-mongodb-aggregation-framework
description: Guides the agent on MongoDB aggregation framework fundamentals, pipeline structure, stage order, and best practices for building powerful server-side data transformations.
---

# MongoDB Aggregation Framework

You are an expert on MongoDB aggregation. When asked about pipeline design or analytics in MongoDB, explain the pipeline model, stage sequencing, common operators, and performance best practices.

## 1. Aggregation is a pipeline, not a query

- The aggregation framework processes documents through an ordered array of stages.
- Each stage takes the documents from the previous stage, transforms them, and passes the result along.
- Use `db.collection.aggregate([stage1, stage2, ...])`.
- Documents are not modified unless a terminal stage like `$out` or `$merge` is included.

### Factory-line metaphor

- `$match` is the input filter station.
- `$group` is the summarization station.
- `$project` / `$set` / `$unset` reshape the output.
- `$sort`, `$skip`, and `$limit` control ordering and volume.

## 2. Why aggregation matters

- Use aggregation for reporting, analytics, and ETL-style transformations.
- It is the engine for dashboard metrics, data cleaning, and server-side joins.
- Aggregation supports full-text and vector search in Atlas with `$search`, `$searchMeta`, and related stages.
- Complex transformations are easier to maintain when broken into small pipeline stages.

## 3. Key stage categories

### Filtering

- `$match` filters documents.
- Equivalent to SQL `WHERE` or `HAVING` when used after grouping.
- Put `$match` as early as possible to reduce document volume.

### Reshaping

- `$project` controls output shape.
- `$set` / `$addFields` add or modify fields.
- `$unset` removes fields.
- Prefer `$set` / `$unset` when most fields should survive unchanged.
- Use `$project` when the output document structure changes significantly.

### Grouping and aggregation

- `$group` accumulates values using operators like `$sum`, `$avg`, `$min`, `$max`, `$count`, and more.
- Group on `_id` for grouping keys.
- Use `$sortByCount()` as a shorthand for grouping plus count+sort.

### Array processing

- `$unwind` flattens arrays into one document per element.
- Filter arrays before `$unwind` when possible to reduce explosion.

### Joining and union

- `$lookup` performs a left outer join with another collection.
- `$unionWith` appends another pipeline or collection result set.

### Output and persistence

- `$out` writes results to a collection and replaces it.
- `$merge` writes into a collection with flexible merge behavior.

## 4. Example pipeline

```js
db.routes.aggregate([
  { $match: { airplane: 'CR2' } },
  { $group: { _id: '$src_airport', totalRoutes: { $sum: 1 } } },
  { $sort: { totalRoutes: -1 } },
  { $limit: 5 }
]);
```

### Explain the example

- `$match` filters to routes with `airplane: 'CR2'`.
- `$group` counts routes by `src_airport`.
- `$sort` orders by highest route count.
- `$limit` returns only the top 5.

## 5. Best practices

- Place `$match` early to reduce data volume.
- Use indexed fields inside `$match` and `$sort` whenever possible.
- Avoid unnecessary `$project` stages; use `$set`/`$unset` if you only add or remove fields.
- Remove unused pipeline stages and merge adjacent stages when safe.
- Monitor with `.explain('executionStats')` to identify bottlenecks.
- Preserve readability by giving pipelines meaningful stage names with comments.

## 6. Tips and tricks

- Use `$sort` before `$limit` to limit only the top documents.
- Use `$count` or `$sortByCount` instead of `$group` + `$sum:1` when counting.
- Use `$addFields` for derived values, then `$unset` to clean up temporary fields.
- For date bucketing, use `$dateTrunc` or `$bucketAuto` instead of application-side grouping.
- When grouping, avoid using large object keys inside `_id` unless necessary.

## 7. Answering style

- When users compare SQL and MongoDB, map early-stage `WHERE` to `$match`, `GROUP BY` to `$group`, and selection to `$project`.
- Clarify that aggregation is a data transformation pipeline, not a CRUD query.
- Provide code snippets in mongosh-style JavaScript.
- Mention Atlas-only search stages when the question is about full-text or vector search.
