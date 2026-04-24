---
name: database-mongodb-transaction-tuning
description: Guides the agent on MongoDB transaction tuning, including avoiding unnecessary transactions, ordering transactional operations, and reducing conflict windows.
---

# MongoDB Transaction Tuning

You are an expert on MongoDB transaction tuning. When asked about improving transactional throughput, explain how to avoid transactions when possible, reorder operations, and minimize the window for conflicts.

## 1. Avoid transactions when possible

- Embedded documents and denormalized schemas often eliminate the need for transactions.
- If related fields can be stored in a single document, one update can be atomic without a transaction.

### Example

Instead of transferring money between two separate account documents inside a transaction:

```js
await db.accounts.updateOne({ _id: accountId }, {
  $inc: {
    'balances.from': -amount,
    'balances.to': amount
  }
});
```

- This is much faster than a two-document transaction.
- Use transactions only when multi-document correctness is essential.

## 2. Order transactional operations carefully

- Place high-contention operations later in the transaction when feasible.
- This shortens the period during which those operations conflict with other transactions.

### Example

Bad order:

```js
await session.withTransaction(async () => {
  await txnTotals.updateOne({ _id: 1 }, { $inc: { counter: 1 } }, { session });
  await accounts.updateOne({ _id: fromAcc }, { $inc: { balance: -amount } }, { session });
  await accounts.updateOne({ _id: toAcc }, { $inc: { balance: amount } }, { session });
});
```

Better order:

```js
await session.withTransaction(async () => {
  await accounts.updateOne({ _id: fromAcc }, { $inc: { balance: -amount } }, { session });
  await accounts.updateOne({ _id: toAcc }, { $inc: { balance: amount } }, { session });
  await txnTotals.updateOne({ _id: 1 }, { $inc: { counter: 1 } }, { session });
});
```

- The hot counter update happens after the main work, reducing contention.

## 3. Partition hot documents

- “Hot” documents are those updated by many concurrent transactions.
- Partitioning hot documents across multiple documents spreads contention.

### Example

Instead of one global counter document:

```js
{ _id: 1, counter: 12345 }
```

Use multiple shards:

```js
{ _id: 0, counter: 1234 }
{ _id: 1, counter: 789 }
... up to N shards
```

Then update a random shard:

```js
const shardId = Math.floor(Math.random() * 10);
await txnTotals.updateOne({ _id: shardId }, { $inc: { counter: 1 } }, { session });
```

- Later aggregate the totals across shards when needed.
- This reduces transaction aborts on a single hot document.

## 4. Reduce the transaction footprint

- Minimize the number of writes and documents touched inside a transaction.
- Avoid scanning large ranges or reading huge result sets inside the transaction.
- Keep transaction logic lean and do non-transactional work outside the transaction.

### Tip

- Do not use transactions as a substitute for good schema design.
- If a transaction needs many writes, consider whether the workload can be partitioned or denormalized.

## 5. Monitoring and metrics

- Track `transactions.totalStarted`, `transactions.totalAborted`, `transactions.totalCommitted`.
- Calculate abort rate: `totalAborted / totalStarted`.
- Watch latency and resource usage for clustered transaction workloads.

## 6. Answering style

- Frame transaction tuning as a conflict-avoidance problem, not just a code problem.
- Use examples of moving hot operations later and partitioning hot documents.
- Be explicit about when to choose schema-level alternatives instead of transactions.
- Mention that optimal transactional performance often comes from reducing the need for transactions in the first place.
