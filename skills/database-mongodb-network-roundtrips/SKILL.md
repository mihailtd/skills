---
name: database-mongodb-network-roundtrips
description: Guides the agent on minimizing MongoDB network round trips through projections, batching, application logic, and deployment topology.
---

# MongoDB Network Round Trips

You are an expert on optimizing MongoDB network round trips. When asked about network performance, explain how to minimize transmissions, pack results efficiently, and keep application code network-friendly.

## 1. Why network round trips matter

- Network transmission time usually exceeds CPU work for database operations.
- Each round trip adds latency, especially across WAN or cross-region links.
- Reducing round trips often yields bigger gains than micro-optimizing server-side operations.

## 2. Pack each response tightly

There are two ways to reduce network overhead:
- Make each document as small as possible.
- Make sure batches contain no empty space.

### Projections

- Projections ensure only required fields are sent over the network.
- Unnecessary fields increase payload size and CPU time on both server and client.

#### Example

```js
db.customers.find({}, { LastName: 1, _id: 0 }).forEach(customer => {
  results[customer.LastName] = (results[customer.LastName] || 0) + 1;
});
```

### Tip

- Always project only the fields you need when reading bulk data.
- For analytics-style queries, even dropping one large field can make a huge difference.

## 3. Batch processing and driver behavior

- MongoDB returns cursor results in batches.
- The default initial batch size is driver-dependent.
- In NodeJS, default `batchSize` may be 1000; in the shell, it is typically much higher.

### Example

```js
const cursor = db.millions.find({}, { n: 1, _id: 0 }).batchSize(10000);
while (await cursor.hasNext()) {
  const doc = await cursor.next();
  count += 1;
}
```

### Tips

- Increase `batchSize` when retrieving many small documents across a slow network.
- Beware: too large a batch size can increase memory use and degrade performance.
- Measure actual driver behavior before committing to a batch-size change.

## 4. Reduce application request churn

- Many small queries cause many round trips.
- Fewer larger queries are often faster, even if they retrieve more data.

### Bad pattern

```js
for (let i = 1; i < max; i++) {
  if (i % 100 === 0) {
    const doc = await db.collection(mycollection).findOne({ _id: i });
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

- The second approach uses fewer network round trips and is often much faster for remote databases.
- The first approach makes dozens or hundreds of separate requests.

## 5. Bulk writes

- Just as reads benefit from batching, writes also benefit from bulk operations.
- Use `insertMany()`, `bulkWrite()`, or batched updates when loading or modifying many documents.
- Bulk writes reduce the number of request/response cycles.

### Example

```js
await db.collection('events').insertMany(batchOfDocs, { ordered: false });
```

## 6. Application proximity and topology

- Co-locate application servers and database servers whenever possible.
- In cloud deployments, choose the same region for application compute and MongoDB Atlas clusters.
- If you cannot co-locate, expect higher per-round-trip latency.

### Tip

- Keep your app code as close as possible to the database gateway.
- For Atlas, choose the nearest available region and shrink the distance between application and cluster.

## 7. Query design for network efficiency

- Use projections in `find()` operations for bulk fetches.
- Use `batchSize()` to tune driver-level network behavior.
- Avoid fetching whole documents if only a few fields are needed.
- Avoid repeated single-document lookups in loops.

## 8. Practical rules of thumb

- If a query reads many documents and only uses a small subset of fields, projection is mandatory.
- If a query fetches most of a collection, batching and network locality are more important than index micro-optimizations.
- Use `cursor` streaming in drivers instead of repeated `findOne()` calls.
- When migrating between environments, compare round-trip latency and adjust batch sizes accordingly.

## 9. Answering style

- Explain network round trips as the dominant cost for remote databases.
- Use the rowboat analogy: fill the boat before crossing.
- Provide concrete code examples for projections, `batchSize()`, and bulk cursor consumption.
- Emphasize driver-specific defaults and the importance of benchmarking.
