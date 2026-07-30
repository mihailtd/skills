---
name: database-mongodb-transaction-performance
description: Guides the agent on MongoDB transaction performance and the cost of MVCC, transaction limits, transient transaction errors, and driver retry behavior.
---

# MongoDB Transaction Performance

You are an expert on MongoDB transaction performance. When asked about transactions, explain the performance trade-offs of snapshot isolation, transaction lifetime, retry behavior, and how MongoDB differs from traditional lock-based SQL transactions.

## 1. Transaction theory and MongoDB

- MongoDB supports ACID transactions with snapshot isolation.
- The snapshot model means each transaction reads a consistent view of the data from the moment it starts.
- MongoDB does not use blocking locks like many SQL systems; instead it uses MVCC and conflict detection.
- This makes MongoDB transactions more concurrency-friendly, but also means they may abort and retry.

### ACID recap

- Atomicity: all operations inside the transaction succeed or none do.
- Consistency: the database moves from one valid state to another.
- Isolation: transactions do not see intermediate results from other in-progress transactions.
- Durability: committed changes survive hardware or OS failures.

## 2. Transaction limits

- MongoDB transactions are limited by default to 60 seconds (`transactionLifetimeLimitSeconds`).
- This limit exists because early MongoDB snapshot history was kept in WiredTiger cache.
- Long-running transactions increase memory pressure and conflict risk.

### Practical guidance

- Keep transactions short and focused.
- Avoid large multi-document or large-result-set operations inside a transaction.
- If a transaction must run longer than 60 seconds, review whether it can be split.

### Tip

- If you see `TransactionTooLarge` or `TransactionTooLong` errors, simplify the transaction or break it into smaller steps.

## 3. Transient transaction errors and retry cost

- MongoDB issues `TransientTransactionError` for conflicts instead of blocking.
- A client must retry the whole transaction when this error occurs.
- The retry cost is high: it discards partial work and starts over.

### Example error

```js
WriteConflict error: this operation conflicted with another operation.
Please retry your operation or multi-document transaction.
```

- Drivers often hide retries automatically, but the server still aborts transactions.
- High abort rates mean poor throughput and long latency.

### Tip

- Monitor `db.serverStatus().transactions` counters: `totalStarted`, `totalAborted`, and `totalCommitted`.
- A high abort rate (e.g. > 10%) is a red flag.

## 4. Driver transaction retry behavior

- Modern MongoDB drivers provide a retry wrapper like `withTransaction()`.
- These wrappers retry on `TransientTransactionError` and `UnknownTransactionCommitResult`.
- Automatic retries simplify code but do not remove the underlying conflict cost.

### Node.js example

```js
await session.withTransaction(async () => {
  await accounts.updateOne({ _id: fromAcc }, { $inc: { balance: -amount } }, { session });
  await accounts.updateOne({ _id: toAcc }, { $inc: { balance: amount } }, { session });
}, transactionOptions);
```

### Python example

```py
with client.start_session() as session:
    def callback(sess):
        accounts.update_one({'_id': from_acc}, {'$inc': {'balance': -amount}}, session=sess)
        accounts.update_one({'_id': to_acc}, {'$inc': {'balance': amount}}, session=sess)
    session.with_transaction(callback, transaction_options)
```

### Tip

- Drivers retry failed transactions transparently, but you should still log retry events and monitor latency.
- Retryable errors are not free; each retry is a full transaction restart.

## 5. When transaction performance is acceptable

- Use transactions for critical multi-document operations where correctness matters more than raw throughput.
- Good candidates include financial transfers, inventory reservations, or multi-collection consistency updates.
- Avoid transactions for simple updates that can be expressed in a single document or via denormalized schemas.

## 6. Answering style

- Emphasize that MongoDB transactions are expensive mostly because of conflict retries.
- Explain how MongoDB’s MVCC snapshot model differs from lock-based SQL transactions.
- Use counters and `serverStatus()` as monitoring examples.
- Recommend short, focused transactions and strong telemetry for retries.
