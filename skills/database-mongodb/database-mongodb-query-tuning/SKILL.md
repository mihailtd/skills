---
name: database-mongodb-query-tuning
description: Guides the agent on MongoDB query tuning, including projections, indexes vs scans, hints, sort optimization, and filter strategies for efficient query plans.
---

# MongoDB Query Tuning

You are an expert on MongoDB query tuning. When asked about query performance, explain how to choose indexes, when to use collection scans, how to optimize sorts, and when to use hints.

## 1. Network-aware query tuning

- Query tuning is not only about CPU and IO; network round trips are often the dominant cost.
- Optimize queries so that the database returns only the fields the application actually needs.
- Use projections aggressively in bulk retrievals.
- Consider the physical location of application servers relative to the database.

## 2. Projections

- Projections reduce the amount of data transmitted across the network and decrease client-side processing.
- A query without projection may transfer entire documents even if only one field is needed.

### Example

```js
db.customers.find({}, { LastName: 1, _id: 0 }).forEach(customer => {
  if (customer.LastName in results) results[customer.LastName]++;
  else results[customer.LastName] = 1;
});
```

### Best practices

- Include projections in `find()` operations whenever fetching bulk data.
- Exclude `_id` explicitly when it is not required.
- If the query uses only a few fields, projecting them can improve throughput by an order of magnitude.

## 3. Batch processing and batchSize

- MongoDB returns results in batches from the server to the client.
- Drivers often default to conservative batch sizes, so tuning `batchSize()` can reduce round trips.
- In the shell, the default batch size is already high; in drivers such as NodeJS, the default may be lower.

### Example

```js
const cursor = db.millions.find({ }, { n: 1, _id: 0 }).batchSize(10000);
while (await cursor.hasNext()) {
  const doc = await cursor.next();
  count += 1;
}
```

### Tips

- Use larger batch sizes when pulling a large number of small documents across a slow network.
- Always benchmark; increasing `batchSize` can hurt performance in some drivers.
- If the driver already uses a large default batch size, manual tuning may not help.

## 4. Avoiding excessive application round trips

- Application code that issues many small queries is often slower than fewer larger queries.
- Pull more documents in one cursor when the access pattern allows it, then apply filtering in application logic.

### Inefficient pattern

```js
for (let i = 1; i < max; i++) {
  if ((i % 100) === 0) {
    const cursor = db.collection(mycollection).find({ _id: i });
    const doc = await cursor.next();
    counter++;
  }
}
```

### Better pattern

```js
const cursor = db.collection(mycollection).find().batchSize(10000);
for await (const doc of cursor) {
  if (doc._id % divisor === 0) counter++;
}
```

- The second approach may read more documents overall, but it often performs much better across high-latency links.
- Favor fewer round trips over fewer index lookups when the network is slow.

## 5. Application architecture

- Locate application servers close to the database to minimize latency.
- In cloud architectures, place your application and MongoDB Atlas cluster in the same region when possible.
- Avoid cross-continent database access for low-latency workloads.

## 6. Indexes versus collection scans

- Indexes are best when a narrow subset of documents is being retrieved.
- If a query accesses most of the collection, a `COLLSCAN` may be faster.
- The break-even point depends on data clustering, cache behavior, document size, and query shape.

### General rules

- For a small percentage of the collection, prefer index scans.
- For all or most of the collection, prefer collection scans.
- Between these extremes, test with actual data.

## 7. Using hints

- Hints override the optimizer and should be used sparingly.
- Use hints when the optimizer chooses an index or scan path that is demonstrably slower than the alternative.
- Use `{$natural: 1}` to force a collection scan.

### Example: force collection scan

```js
const exp = db.customers.explain('executionStats')
  .find({ dateOfBirth: { $gt: new Date('1900-01-01') } })
  .hint({ $natural: 1 });
```

### Example: force index

```js
const exp = db.customers.explain('executionStats')
  .find({ Country: 'India', dateOfBirth: { $gt: new Date('1990-01-01') } })
  .hint({ dateOfBirth: 1 });
```

### Warning

- Hints can prevent MongoDB from using new indexes or runtime plan improvements.
- Use hints only after careful testing and as a last resort.

## 8. Sort optimization

- Sorts are most efficient when supported by an index.
- Without an index, MongoDB performs a blocking sort, which can use a lot of memory and delay first-row delivery.

### Example of blocking sort plan

```js
const plan = db.customers.explain().find().sort({ dateOfBirth: 1 });
```

- If the sort is indexed, the plan becomes `IXSCAN` followed by `FETCH`.

### Supporting filter + sort

- Create compound indexes with filter keys first and sort keys next.

```js
db.customers.createIndex({ Country: 1, dateOfBirth: 1 });
```

### Tip

- Use an index for fast retrieval of the first few sorted documents.
- If you need the entire sorted set, a blocking sort may sometimes be faster.

## 9. Perfect coverage

- The ideal index contains all filter fields, sort fields, and optionally projected fields.
- A perfectly covered query can return results from the index without fetching documents.

### Example

```js
db.customers.createIndex(
  { Country: 1, 'views.title': 1, LastName: 1, Phone: 1 },
  { name: 'CntTitleLastPhone_ix' }
);

const exp = db.customers.explain('executionStats')
  .find({ Country: 'Japan', 'views.title': 'MUSKETEERS WAIT' }, { Phone: 1, _id: 0 })
  .sort({ LastName: 1 });
```

- If the plan shows `PROJECTION_COVERED`, the query is fully served by the index.
- If `FETCH` processes the same number of documents as `IXSCAN`, the index retrieved all matched documents efficiently.

## 10. Filter strategies

### $ne queries

- `$ne` can use indexes, but the scan may cover most of the index.
- For broad `$ne` filters, a collection scan may be better.

### Range queries

- Use indexes for narrow ranges.
- If the range covers most of the collection, prefer a collection scan.
- Example:

```js
const exp = db.iotData.explain('executionStats').find({ a: { $gt: 0 } });
```

- Use `hint({ $natural: 1 })` when the indexed range scan is slower than a scan.

### $or and $in

- `$or` on multiple indexed fields can perform index merge or OR execution.
- Index all attributes in an `$or` to avoid collection scans.

### Array queries

- Indexed array field queries such as `$all`, `$elemMatch`, and equality against array fields can use index scans.
- `$size` cannot use a normal array index and usually results in a collection scan.

### Regular expressions

- Anchor regex patterns with `^` to use an index efficiently.
- Use case-insensitive collation indexes for efficient `$regex /.../i` queries.

### $exists

- `$exists: true` can use an index, but it may scan every indexed key.
- A sparse index can optimize `$exists: true` queries, but not `$exists: false`.

## 11. Collection scan optimization

- If a collection scan is unavoidable, reduce collection size.
- Move large infrequently accessed fields to separate collections (vertical partitioning).
- Use sharding to parallelize scans across shards if the dataset is large.
- Compact collections only when disk bloat is a real issue and downtime is acceptable.

## 12. Answering style

- Focus on practical tuning actions with code examples.
- Explain why a plan was chosen and when an alternate plan is better.
- Use terms like `IXSCAN`, `COLLSCAN`, `FETCH`, `SORT`, `PROJECTION_COVERED`, and `hint()`.
- Emphasize testing with `explain('executionStats')` and driver-specific batch-size behavior.
