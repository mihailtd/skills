---
name: database-mongodb-indexing
description: Guides the agent on MongoDB indexing fundamentals, B-tree behavior, selectivity, compound and covering indexes, index scans, partial/sparse indexes, and index maintenance tradeoffs.
---

# MongoDB Indexing

You are an expert on MongoDB indexes. When asked about query performance, indexing strategy, or index types, explain how MongoDB indexes work, when to create them, and how they trade off read speed for write overhead.

## 1. What is an index in MongoDB?

- An index is a separate on-disk data structure that allows MongoDB to locate documents without scanning the entire collection.
- MongoDB’s default index structure is a B-tree.
- Each collection has an implicit unique index on `_id`.
- Indexes improve read performance, but they impose overhead on inserts, updates, and deletes.

## 2. B-tree indexes and how they work

### Anatomy of a B-tree index

- Header block: top-level entry point.
- Branch blocks: route searches to leaf blocks.
- Leaf blocks: contain key values and pointers to documents.
- Leaf blocks are doubly linked to support range scans in both directions.

### Why B-trees are efficient

- The tree depth is typically shallow: header, branch, leaf.
- Querying a single value usually requires only 1–2 disk reads after in-memory branches are traversed.
- Range queries are efficient because leaf blocks link sequentially.

### Index maintenance cost

- Inserts require finding the leaf block and writing a new entry.
- If the leaf block is full, MongoDB splits the block and updates parent branches.
- Updates that modify indexed fields must update the index.
- Deletes remove index entries.
- Index maintenance makes inserts/updates/deletes more expensive.

> Rule: indexes speed reads, but slow writes.

## 3. Selectivity and uniqueness

### Selectivity

- Selectivity measures how many documents share the same key value.
- Highly selective indexes have many unique values and narrow the result set.
- Low-selectivity indexes (e.g. `gender`) are less useful on their own.

### Unique indexes

- A unique index prevents duplicate values for the indexed key.
- Unique indexes are usually very selective and efficient.
- MongoDB automatically creates a unique index on `_id`.

## 4. Index scans and range queries

- `IXSCAN` means MongoDB uses an index scan.
- Range scans are possible because leaf blocks are linked.
- Example range query:

```js
db.customers.find({
  dateOfBirth: { $gt: ISODate('1980-01-01'), $lt: ISODate('1990-01-01') }
});
```

- With an index on `dateOfBirth`, MongoDB scans from the lower bound until the upper bound.
- `explain()` output includes `indexBounds` showing the scanned range.

### When index scans are bad

- If the scanned range is too wide, an index scan may be slower than `COLLSCAN`.
- The optimizer decides based on estimated work.
- Use `explain('executionStats')` to see whether the index scan is efficient.

## 5. Compound indexes

- A compound index includes multiple fields.
- It is usually more selective than single-field indexes.
- It can support queries on any prefix of its key pattern.

### Leading prefix rule

- An index on `{ LastName: 1, FirstName: 1, dateOfBirth: 1 }` can support:
  - `LastName`
  - `LastName` + `FirstName`
  - `LastName` + `FirstName` + `dateOfBirth`
- It cannot support queries on `FirstName` alone or `dateOfBirth` alone.
- Put frequently filtered fields first.

### Selectivity and order

- The most selective field is often a good candidate for the leading position.
- However, if a less selective field appears in many queries alone, keep it earlier.
- Use the query workload to order compound index keys.

## 6. Covering indexes

- A covered query is one where all queried fields and projected fields are present in the index.
- MongoDB can answer the query entirely from the index without fetching documents.
- Example:

```js
db.people.find(
  { LastName: 'HENNING', FirstName: 'ALBERTO', dateOfBirth: ISODate('1953-12-23') },
  { _id: 0, Phone: 1 }
).sort({ Phone: 1 });
```

- If the index is `{ LastName: 1, FirstName: 1, dateOfBirth: 1, Phone: 1 }`, the query can become covered.

## 7. Index merges and intersections

### Index intersection

- MongoDB can intersect multiple single-field indexes for `$and` conditions.
- Example plan:
  - `IXSCAN a_1`
  - `IXSCAN b_1`
  - `AND_SORTED`
  - `FETCH`
- A compound index is usually better for `$and` conditions.

### Index merge for `$or`

- `$or` expressions often use multiple indexes and merge results.
- Example plan:
  - `IXSCAN a_1`
  - `IXSCAN b_1`
  - `OR`
  - `FETCH`
  - `SUBPLAN`
- Merged indexes are useful when a compound index is impractical.

## 8. Partial and sparse indexes

### Partial indexes

- Partial indexes index only documents that match a filter expression.
- They are smaller and cheaper to maintain than full indexes.
- Example:

```js
db.tweets.createIndex(
  { 'user.name': 1, retweet_count: 1 },
  { partialFilterExpression: { retweet_count: { $gt: 0 } } }
);
```

- Queries must include the partial filter condition to use the index.
- Partial indexes are ideal for active or recent subsets of data.

### Sparse indexes

- Sparse indexes only index documents that contain the indexed field.
- They ignore documents where the field is missing.
- Example:

```js
db.customers.createIndex({ runtime: 1 }, { sparse: true });
```

- A sparse index cannot support `$exists: false` efficiently.
- It is useful when a field is optional and only some documents use it.

## 9. Index planning for sorting and joins

### Sorting

- MongoDB can use an index to avoid an in-memory sort if the sort keys align with the index.
- Add sort fields to the compound index after filter fields.
- Avoid explicit `SORT` stages by supporting the sort with an index.

### Joins

- `$lookup` and `$graphLookup` should use indexed join keys.
- Join stages can be expensive if the foreign collection is not indexed.
- Use `$lookup` with `localField`/`foreignField` or `pipeline` + `let` on indexed fields.

## 10. Index overhead and maintenance

- Every insert, update, or delete must update all relevant indexes.
- Indexes on frequently updated fields are especially costly.
- If a field is updated frequently, prefer not to index it unless necessary.
- A high insert/delete workload with many indexes can degrade throughput.

### Practical advice

- Create only indexes that improve important queries.
- Monitor index usage and remove unused indexes.
- Avoid indexing supersets of frequently changing fields.
- Remember: a document may be inserted once but updated many times.

## 11. Tips and tricks

- Use `db.collection.getIndexes()` to review existing indexes.
- Use `explain('executionStats')` to verify whether a query uses an index.
- If `IXSCAN` shows a huge range of keys, consider whether the index is selective enough.
- If `SORT` remains in the winning plan, add a supporting index for the sort order.
- Prefer compound indexes for queries that filter on multiple fields together.
- Use `partialFilterExpression` to limit index size for hot subsets of data.
- Use `sparse` when a field is optional and only the indexed documents matter.
- Don’t create indexes for every field; each index costs writes.
- If an index has low use and high maintenance cost, drop it.

## 12. Answering style

- Explain indexing as performance tuning, not schema definition.
- Use B-tree vocabulary: header, branch, leaf, split, index scan.
- Mention the write cost of index maintenance.
- Provide example commands for index creation, partial indexes, and coverage.
- Recommend `explain()` as the verification step after creating an index.
