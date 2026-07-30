---
name: database-mongodb-bulk-operations
description: Guides the agent on MongoDB bulk write operations, including bulkWrite, ordered versus unordered execution, batch splitting, and cross-collection writes in MongoDB 8.0.
---

# MongoDB Bulk Operations

You are an expert on MongoDB bulk writes. When asked about writing many documents or mixed write operations, explain `bulkWrite()`, batch limits, ordered/unordered semantics, error handling, and the new MongoDB 8.0 multi-collection bulk API.

## 1. db.collection.bulkWrite()

- Use `bulkWrite()` when you need to combine inserts, updates, replacements, and deletes into a single collection-specific request.
- Each operation is expressed as an object in an array.

Example:

```js
await db.routes.bulkWrite([
  { insertOne: { document: { airline: { id: 413, name: 'American Airlines' }, src_airport: 'DFW', dst_airport: 'LAX', airplane: '737' } } },
  { updateOne: { filter: { 'airline.id': 411 }, update: { $set: { airplane: 'A380' } } } },
  { deleteOne: { filter: { 'airline.id': 412, src_airport: 'CDG' } } },
]);
```

## 2. Ordered and unordered behavior

- Default is ordered execution.
  - Later operations are not attempted after an error.
- `ordered: false` allows processing to continue after failures.

Example:

```js
await db.routes.bulkWrite(operations, { ordered: false });
```

### Important differences

- Ordered bulk writes preserve operation order and stop on the first error.
- Unordered bulk writes can run faster but may complete operations in a different order.
- Unordered mode is useful when you care about throughput more than exact sequencing.

## 3. Batch splitting and limits

- MongoDB drivers automatically split long bulk arrays when necessary.
- `maxWriteBatchSize` defaults to 100,000 operations.
- If your write payload exceeds the limit, the driver divides it into smaller batches.
- Large document sizes still require each document to be under the 16 MB BSON limit.

### Tip

- Do not build massive bulk arrays in memory without chunking; use a streaming or generator pattern when possible.

## 4. New multi-collection bulkWrite in MongoDB 8.0

- MongoDB 8.0 introduced a `bulkWrite` command that can operate across multiple collections in a single request.
- This is done via `db.adminCommand()` with `bulkWrite: 1`.
- The command uses `ops` for operations and `nsInfo` to declare namespaces.

Example:

```js
await db.adminCommand({
  bulkWrite: 1,
  ops: [
    {
      insert: 0,
      document: {
        airline: { id: 413, name: 'American Airlines', alias: 'AA', iata: 'AAL' },
        src_airport: 'DFW',
        dst_airport: 'LAX',
        airplane: '737',
      },
    },
    {
      insert: 1,
      document: {
        accounts: [371138, 324287],
        tier_and_details: {
          '0df078f33aa74a2e9696e0520c1a828a': { tier: 'Bronze', id: '0df078f33aa74a2e9696e0520c1a828a', active: true },
        },
      },
    },
  ],
  nsInfo: [
    { ns: 'sample_training.routes' },
    { ns: 'sample_analytics.customers' },
  ],
});
```

### Tip

- Use the multi-collection bulk API only when you need a single coordinated write request across namespaces.
- For normal application logic, prefer collection-scoped `bulkWrite()` and simple retry/error handling.

## 5. Error handling

- Bulk operations can return partial success.
- Inspect `result.hasWriteErrors()` or error documents to identify failed operations.
- In unordered mode, multiple operation failures may be reported.
- When one operation fails in ordered mode, later operations are skipped.

## 6. Performance considerations

- Bulk operations are best for large insert/update/delete workloads.
- Avoid bulk writes with too many large documents in a single array.
- If you bulk insert into an indexed collection with random keys, index maintenance can slow writes.
- Consider dropping or deferring index creation for massive writes, then rebuild the index afterward.

## 7. Answering style

- Explain when to use `bulkWrite()` versus repeated individual operations.
- Be explicit about ordered/unordered semantics and batch limits.
- Prefer examples that show mixed writes or multi-collection 8.0 commands when the user asks about advanced bulk operations.
- Always call out the difference between collection-specific `bulkWrite()` and the new admin-level multi-collection bulk API.
