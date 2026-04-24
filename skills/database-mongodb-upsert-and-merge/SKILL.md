---
name: database-mongodb-upsert-and-merge
description: Guides the agent on MongoDB upserts, aggregation-based bulk upserts, cloning data with $merge, and avoiding unnecessary round trips.
---

# MongoDB Upsert and Merge Patterns

You are an expert on MongoDB upserts and server-side merge patterns. When asked about conditional inserts or bulk document cloning, explain how to use `upsert`, `$merge`, and aggregation-based writes to reduce round trips and improve throughput.

## 1. Upsert fundamentals

- `upsert: true` performs an update if a match exists, otherwise inserts a new document.
- It avoids the two-step `find` + `insert/update` pattern.
- Upserts are useful when you do not know whether the target document exists.

### Example

```js
await db.target.updateOne(
  { _id: doc._id },
  { $set: doc },
  { upsert: true }
);
```

### Tip

- Ensure the filter fields used for upsert are backed by an index to avoid unnecessary scanning.
- Avoid using upsert with broad or expensive filters.

## 2. Why upsert beats manual existence checks

### Bad pattern

```js
db.source.find().forEach(doc => {
  const count = db.target.count({ _id: doc._id });
  if (count === 0) {
    db.target.insert(doc);
  } else {
    db.target.update({ _id: doc._id }, { $set: doc });
  }
});
```

- This performs extra network round trips and duplicate lookup work.

### Better pattern

```js
db.source.find().forEach(doc => {
  db.target.update(
    { _id: doc._id },
    { $set: doc },
    { upsert: true }
  );
});
```

- One command per document instead of two.
- Much faster over remote networks.

## 3. Bulk upsert with aggregation `$merge`

- If the source data is already in MongoDB, use `$merge` to perform bulk upserts inside the server.
- `$merge` can insert new documents and update existing documents in one aggregation.
- This avoids pulling documents into the application and re-pushing them.

### Example

```js
db.source.aggregate([
  { $merge: {
      into: 'target',
      on: '_id',
      whenMatched: 'replace',
      whenNotMatched: 'insert'
    }
  }
]);
```

### Tip

- Use `$merge` for bulk upsert workloads where the source is a collection.
- This is often much faster than looping over documents in application code.

## 4. Cloning and copying data without network hops

- Use aggregation with `$merge` to clone documents selected from one collection into another.
- This keeps the data movement inside the server.

### Example

```js
const newOrderId = db.orders.insertOne({ /* ... */ }).insertedId;

db.lineitems.aggregate([
  { $match: { orderId: oldOrderId } },
  { $project: { _id: 0, orderId: 0 } },
  { $addFields: { orderId: newOrderId } },
  { $merge: { into: 'lineitems' } }
]);
```

- This is faster than fetching documents to the app and re-inserting them.
- It also reduces client-side network bandwidth and latency.

## 5. $out vs $merge

- `$out` writes aggregation results to a target collection, replacing it entirely.
- `$merge` writes results into a target collection using upsert-style semantics.
- Use `$merge` when you need incremental insert/update behavior.
- Use `$out` for wholesale replacement of a materialized result set.

## 6. Answering style

- Emphasize single-operation upsert semantics versus manual existence checks.
- Show examples of `upsert: true` and server-side `$merge` pipelines.
- Highlight network savings from avoiding round trips.
- State when `$merge` is better than looping in application code.
