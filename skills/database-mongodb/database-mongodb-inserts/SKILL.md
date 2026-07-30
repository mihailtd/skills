---
name: database-mongodb-inserts
description: Guides the agent on MongoDB insert operations, including insertOne, insertMany, ordered/unordered writes, custom _id values, batch sizing, and error handling.
---

# MongoDB Inserts

You are an expert on MongoDB insert operations. When asked about adding new documents, explain the differences between `insertOne()` and `insertMany()`, how ordered and unordered batches behave, and how to avoid common size and index-performance traps.

## 1. insertOne() vs insertMany()

- Use `insertOne()` for a single document.
- Use `insertMany()` when you need to insert multiple documents in one logical operation.
- `insertMany()` is usually much faster than repeated `insertOne()` calls because it reduces round trips and uses bulk write semantics.

Example:

```js
const result = await db.routes.insertOne({
  airline: { id: 410, name: 'Lufthansa', alias: 'LH', iata: 'DLH' },
  src_airport: 'MUC',
  dst_airport: 'JFK',
  codeshare: '',
  stops: 0,
  airplane: 'A380',
});
console.log(result.acknowledged, result.insertedId);
```

```js
const result = await db.routes.insertMany([
  {
    airline: { id: 413, name: 'American Airlines', alias: 'AA', iata: 'AAL' },
    src_airport: 'DFW',
    dst_airport: 'LAX',
    codeshare: '',
    stops: 0,
    airplane: '737',
  },
  {
    airline: { id: 411, name: 'British Airways', alias: 'BA', iata: 'BAW' },
    src_airport: 'LHR',
    dst_airport: 'SFO',
    codeshare: 'Y',
    stops: 0,
    airplane: '747',
  },
]);
console.log(result.acknowledged, result.insertedIds);
```

## 2. Custom _id values

- MongoDB generates an `ObjectId` automatically when `_id` is absent.
- You can supply your own `_id` if you need a custom natural key or deterministic identifier.
- Do not supply duplicate `_id` values or the insert will fail with a duplicate key error.

Example:

```js
await db.routes.insertOne({
  _id: 'route-123',
  airline: { id: 410, name: 'Lufthansa' },
  src_airport: 'MUC',
  dst_airport: 'JFK',
});
```

## 3. Ordered vs unordered insertMany()

- Default behavior is `ordered: true`.
  - If one document fails, MongoDB stops processing the remaining documents.
  - This preserves input order and provides a deterministic error point.
- Use `ordered: false` to continue inserting after a failure.
  - This can improve throughput for large inserts.
  - The returned inserted document order may differ from the input order.

Example:

```js
await db.routes.insertMany(docs, { ordered: false });
```

### Error handling tips

- For `ordered: false`, inspect the error details to identify failed indexes.
- The error response includes the index of the failed document in the original array.
- Do not rely on returned insert order when `ordered: false`.
- When inserting mixed valid/invalid documents, capture both successes and failures.

## 4. Size and batch limits

- MongoDB limits each BSON document to 16 MB.
- For `insertMany()`, the combined batch payload must also fit within server limits.
- The driver splits large bulk inserts into smaller batches when necessary.
- `maxWriteBatchSize` defaults to 100,000 operations.
  - A batch of 300,000 operations will be split into three 100,000-operation batches.

### Practical guideline

- Keep individual documents small.
- Monitor document size when inserting arrays, embedded documents, or large strings.
- Use `insertMany()` for thousands of documents, but chunk into batches if documents are large.

## 5. Write-performance considerations

- Inserts on indexed fields are slower because the index must be updated for each inserted document.
- For random values in an indexed field (especially hashed indexes), performance can decline sharply.
- If you are inserting a large number of random or bulk documents:
  - consider dropping the index before the bulk insert and rebuilding it afterward,
  - or insert into an unindexed collection and create the index post-insert.
- This is appropriate only when you have downtime or can tolerate slower reads during rebuild.

## 6. Acknowledged writes and oplog behavior

- `insertOne()` and `insertMany()` both return `acknowledged: true` when successful.
- Each inserted document is recorded in the oplog if the operation succeeds.
- Failed inserts do not create oplog entries.

## 7. Answering style

- Show the exact method signature and the recommended options object.
- Emphasize `insertMany()` for bulk inserts and `insertOne()` for single documents.
- Call out ordered/unordered behavior and error reporting clearly.
- Use concrete examples with airline route documents when possible.
- Mention the 16 MB BSON limit and the 100,000-operation batch limit as hard constraints.
