---
name: database-mongodb-delete-optimization
description: Guides the agent on MongoDB delete optimization, logical deletes, index impact, and strategies for handling high-volume delete workloads.
---

# MongoDB Delete Optimization

You are an expert on MongoDB delete operations and deletion strategies. When asked about removing documents, explain how delete performance is affected by indexes, how to use logical deletes, and when to avoid physical deletes in hot workloads.

## 1. Delete performance fundamentals

- Deletes must update all indexes on the collection.
- Heavy indexing increases the cost of delete operations.
- A delete is as expensive as the number of indexes that must be maintained.

### Tip

- Avoid unnecessary indexes on high-delete collections, especially for streaming or transient data.
- Keep delete workloads in mind when evaluating index additions.

## 2. Filter tuning for deletes

- Use targeted delete filters backed by indexes.
- A broad delete filter may require a collection scan and can be very expensive.

### Example

```js
db.orders.deleteMany({ orderStatus: 'cancelled', createdAt: { $lt: ISODate('2025-01-01') } });
```

- If `orderStatus` and `createdAt` are indexed, the delete can locate affected rows efficiently.

## 3. Logical deletes

- For heavy delete workloads, consider logical delete flags instead of physical removal.
- Logical delete means setting a field such as `deleted: true` or `deletedAt: ISODate()`.
- This avoids repeated index maintenance on the deleted documents.

### Example

```js
await db.customers.updateMany(
  { status: 'removed' },
  { $set: { deleted: true, deletedAt: new Date() } }
);
```

### Tip

- If you use logical deletes, include the delete flag in all collection queries and relevant indexes.
- Example index: `{ deleted: 1, status: 1, createdAt: -1 }`.

## 4. Periodic physical cleanup

- Logical deletes should be followed by periodic physical cleanup during maintenance windows.
- Use a low-priority bulk delete or a background process to purge logically deleted documents.
- This keeps the collection size manageable without impacting peak workloads.

## 5. Delete explain and diagnostics

- Explain `deleteOne()` and `deleteMany()` to see whether MongoDB uses an index.
- If the explain plan shows `COLLSCAN`, the delete filter may be unindexed.

### Example

```js
const exp = db.customers.explain('executionStats').deleteMany({ deleted: true });
printjson(exp);
```

- A well-indexed delete shows `IXSCAN` instead of `COLLSCAN` in the explain plan.

## 6. Write concern and delete latency

- Delete operations obey write concern settings the same way inserts and updates do.
- Stronger write concern increases latency and reduces throughput.
- Do not weaken write concern unless you have explicitly accepted the durability trade-offs.

## 7. Answering style

- Explain delete cost in terms of index maintenance.
- Recommend targeted filters and indexed delete predicates.
- Define logical delete clearly and mention the need to account for the delete flag in queries and indexes.
- Use examples showing both physical and logical deletes.
