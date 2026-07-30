---
name: database-mongodb-transaction-hotspot-management
description: Guides the agent on MongoDB transaction hotspot management, including transaction conflicts, hot document partitioning, and strategic payload distribution.
---

# MongoDB Transaction Hotspot Management

You are an expert on MongoDB transaction hotspot management. When asked about write contention and transaction conflicts, explain how to identify hot documents, partition them, and avoid high abort rates.

## 1. What makes a document hot?

- A document is hot when many concurrent transactions try to read or write it.
- Hot documents increase `TransientTransactionError` rates.
- Common hot-document patterns include global counters, status flags, and frequently updated summary records.

### Tip

- If one document is updated by most transactions, it is a likely hotspot.
- Use server-side metrics to identify this pattern.

## 2. Avoid single global counters in transactions

- Global counters are classic hot spots.
- If every transaction updates the same counter document, transactions will conflict.

### Better design

- Split the counter across multiple documents.
- Example shard keys:

```js
{ _id: 0, counter: 1200 }
{ _id: 1, counter: 1100 }
...
```

- Update a random shard inside the transaction:

```js
const bucket = Math.floor(Math.random() * 10);
await txnTotals.updateOne({ _id: bucket }, { $inc: { counter: 1 } }, { session });
```

- Periodically aggregate the shards for a global total.

## 3. Use strategic partitioning

- Partition hot documents by tenant, region, time window, or workload bucket.
- This reduces the likelihood that concurrent transactions target the same document.

### Example

For a per-day write log:

```js
{ _id: { day: '2025-04-24', bucket: 3 }, count: 124 }
```

- Use a shard key that spreads concurrent writes.
- Prefer a high-cardinality field for partitioning.

## 4. Detecting contention patterns

- Monitor transaction abort counters and check logs for `WriteConflict` events.
- If `txnCounts().totalAborted` is high, transaction contention is likely.

### Tip

- High abort percentages with short transactions usually indicate hot document contention.
- If aborts occur mostly on one type of operation, review the document access pattern.

## 5. Optimize access order in transactions

- Place low-contention reads and writes first.
- Place high-contention updates later.
- This reduces the window during which hot document conflicts can abort the transaction.

### Example

Good order:

```js
await session.withTransaction(async () => {
  await accounts.updateOne({ _id: fromAcc }, { $inc: { balance: -amount } }, { session });
  await accounts.updateOne({ _id: toAcc }, { $inc: { balance: amount } }, { session });
  await stats.updateOne({ _id: shardId }, { $inc: { counter: 1 } }, { session });
});
```

- Defer the hot counter or summary document update until the end.

## 6. Use non-transactional summaries when possible

- If a summary or log can tolerate eventual consistency, write it outside the transaction.
- This eliminates one source of transaction conflicts.

### Example

- Perform the business-critical transfer inside a transaction.
- Emit analytics or audit events asynchronously outside the transaction.

## 7. Answering style

- Focus on document-level contention and how it causes transaction retries.
- Use partitioning and bucketization as the main mitigation strategies.
- Include examples of global counters and per-shard counters.
- Mention that not all problems are solved by bigger clusters; often the schema needs to change.
