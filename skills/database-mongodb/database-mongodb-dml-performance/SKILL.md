---
name: database-mongodb-dml-performance
description: Guides the agent on MongoDB data manipulation performance, including insert/update/delete filtering, index overhead, explain() for DML, write concern, and workload-driven tuning.
---

# MongoDB Data Manipulation Performance

You are an expert on MongoDB insert/update/delete performance. When asked about data manipulation tuning, describe how to optimize filters, account for index maintenance, use explain() on DML, and avoid over-indexing.

## 1. Filter optimization for updates and deletes

- All update and delete performance starts with the filter.
- Use the same query-tuning principles as for `find()` and aggregation `$match`.
- Index the fields that appear in the filter condition whenever possible.

### Tip

- If an update or delete includes a filter condition, make sure the filter is covered by an efficient index.
- A poorly filtered delete or update can scan the entire collection and do a lot of needless work.

## 2. Explain DML statements

- You can run `explain()` on `update()` and `delete()` statements.
- The explain output shows how MongoDB will locate the documents to modify or remove.
- Use `executionStats` to see whether the statement would scan the collection or use an index.

### Example

```js
const exp = db.customers.explain('executionStats').update(
  { viewCount: { $gt: 50 } },
  { $set: { discount: 10 } },
  { multi: true }
);
printjson(exp);
```

- If the plan shows `COLLSCAN`, the update is scanning the full collection.
- If the plan shows `IXSCAN`, the update is using an index to locate matching documents.

### Tip

- Use `explain()` before running large updates or deletes to avoid accidental full collection scans.

## 3. Index maintenance overhead

- Every insert or delete must update all relevant indexes on the collection.
- Every update that changes an indexed field also updates that index.
- Indexes on frequently mutated fields are particularly expensive.

### Practical rule

- A document can be inserted or deleted once, but may be updated many times.
- Avoid indexing fields that are updated frequently unless the index is critical for query performance.

### Tip

- Use indexes selectively: only keep indexes that pay off for queries, not just because they seem useful.

## 4. Finding unused indexes

- Use `$indexStats` to track index access counts.
- Unused indexes still impose write overhead.

### Example

```js
db.customers.aggregate([
  { $indexStats: {} },
  { $project: { name: 1, 'accesses.ops': 1 } }
]);
```

- If an index reports `accesses.ops: 0` over a reasonable monitoring window, consider dropping it.
- Exceptions:
  - Unique indexes that enforce data integrity.
  - TTL indexes that expire documents.

### Tip

- Do not drop audit, uniqueness, or TTL indexes just because they report low usage.
- If the server was restarted recently, stats may be misleading.

## 5. Write concern and durability trade-offs

- Write concern affects the latency and throughput of inserts, updates, and deletes.
- `w: 1` is faster than `w: majority`, but less durable.
- Higher write concern means waiting for more replica acknowledgments.

### Tip

- Never sacrifice data integrity for a small performance gain without understanding the risks.
- Adjust write concern only when you have a clear reliability requirement and know the cluster topology.

## 6. Answering style

- Connect DML performance to query performance: fast filters reduce DML work.
- Call out index maintenance as the key extra cost for inserts, updates, and deletes.
- Mention `explain()` on DML and `$indexStats` as diagnostic tools.
- Use concrete examples and emphasize workload-driven decision making.
