---
name: database-mongodb-transactions
description: Guides the agent on MongoDB ACID transactions, WiredTiger internals, transaction APIs, driver examples, and transaction best practices for Node.js, Python, and Ruby.
---

# MongoDB Transactions and ACID

You are an expert on MongoDB multidocument transactions. When asked about transaction behavior, storage engine guarantees, retry logic, or driver usage, explain the underlying concepts clearly, provide code snippets, and highlight production-ready patterns.

## 1. MongoDB transactions are ACID-enabled

- MongoDB provides ACID behavior for multidocument transactions on replica sets and sharded clusters.
- Single-document operations are already atomic; transactions extend atomicity across multiple documents, collections, and databases.
- Transaction properties:
  - Atomicity: all operations succeed or all are rolled back.
  - Consistency: the database moves from one valid state to another.
  - Isolation: transaction reads and writes are isolated from other concurrent operations.
  - Durability: committed transactions survive crashes thanks to write-ahead logging and checkpoints.

## 2. Why WiredTiger matters

- WiredTiger is MongoDB’s default storage engine.
- It uses MVCC for concurrency, allowing readers and writers to operate concurrently.
- Journaling plus checkpoints ensure durability.
- WiredTiger maintains snapshots and supports fine-grained MVCC history for transaction isolation.

### Storage engine details

- Snapshots: MongoDB transactions read from a consistent snapshot created at transaction start.
- Checkpoints: scheduled every 60 seconds, but the previous checkpoint remains valid during a new checkpoint.
- Journaling: write-ahead log records writes between checkpoints; replay is used after crashes.
- Compression: WiredTiger compresses collection blocks (Snappy default) and index blocks (prefix compression). You can choose zlib or zstd for collections.

### Memory and caching

- WiredTiger cache default is max(50% of RAM minus 1 GB, 256 MB).
- On 8 GB RAM, default cache is 3.5 GB.
- Cache is critical for index and transaction performance.

## 3. Transaction models and APIs

MongoDB exposes two transaction APIs in drivers:

### Core API

- Manual: `startTransaction()`, run operations, then `commitTransaction()` or `abortTransaction()`.
- Requires explicit retry logic for `TransientTransactionError` and `UnknownTransactionCommitResult`.
- Good for learning or low-level control.

### Callback API (recommended)

- The driver manages begin/commit/abort and retry logic.
- Use `withTransaction()` or equivalent.
- It handles retryable error labels automatically.
- Preferred in production for simpler code.

## 4. Transaction best practices

### Read/write concerns

- Use `readConcern: { level: 'snapshot' }` inside transactions.
- Use `writeConcern: { w: 'majority' }` for durability.
- Use `readPreference: 'primary'` for transaction work.

### Keep transactions short

- Avoid transactions longer than 60 seconds.
- Break large workflows into smaller steps.
- Long transactions increase lock contention and retry pressure.

### Maximum work per transaction

- Limit to around 1,000 modified documents.
- Avoid large batch operations inside a single transaction where possible.

### Error handling

- Retry on `TransientTransactionError`.
- Retry commits on `UnknownTransactionCommitResult`.
- Abort on nonretryable errors.
- Always end the session after the transaction.

### Use transactions only when needed

- Prefer embedded documents and denormalized data.
- Use transactions for business-critical multi-document changes, not as a default.

## 5. Transaction examples

### Mongosh manual transaction example

```js
function executeTransaction(session) {
  const db = session.getDatabase('sample_analytics');
  session.startTransaction({
    readConcern: { level: 'snapshot' },
    writeConcern: { w: 'majority' },
    readPreference: 'primary'
  });

  try {
    const account = db.accounts.findOne({ account_id: 371138 }, { session });
    if (!account) throw new Error('Account not found');

    db.accounts.updateOne(
      { account_id: 371138 },
      {
        $set: { limit: 12000, last_transaction_date: new Date() },
        $inc: { transaction_count: 1 }
      },
      { session }
    );

    db.customers.updateMany(
      { accounts: { $in: [371138] } },
      {
        $set: { last_transaction_date: new Date() },
        $inc: { transaction_count: 1 }
      },
      { session }
    );

    db.transactions.insertOne({
      account_id: 371138,
      transaction_count: (account.transaction_count || 0) + 1,
      date: new Date(),
      amount: 1500,
      transaction_code: 'buy'
    }, { session });

    session.commitTransaction();
  } catch (err) {
    session.abortTransaction();
    throw err;
  }
}
```

### Mongosh retry wrapper

```js
function runTransactionWithRetry() {
  const maxRetries = 5;
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const session = db.getMongo().startSession();
    try {
      executeTransaction(session);
      return;
    } catch (error) {
      if (error.errorLabels &&
          (error.errorLabels.includes('TransientTransactionError') ||
           error.errorLabels.includes('UnknownTransactionCommitResult'))) {
        continue;
      }
      throw error;
    } finally {
      session.endSession();
    }
  }
  throw new Error('Max retries reached. Transaction failed.');
}
```

### Node.js `withTransaction` example

```js
const { MongoClient } = require('mongodb');
const uri = 'your_mongodb_connection_string';

async function run(accountId) {
  const client = new MongoClient(uri);
  await client.connect();
  const session = client.startSession();

  const transactionOptions = {
    readConcern: { level: 'snapshot' },
    writeConcern: { w: 'majority' },
    readPreference: 'primary'
  };

  try {
    await session.withTransaction(async () => {
      const accounts = client.db('sample_analytics').collection('accounts');
      const customers = client.db('sample_analytics').collection('customers');
      const transactions = client.db('sample_analytics').collection('transactions');
      const currentDate = new Date();

      const account = await accounts.findOne({ account_id: parseInt(accountId, 10) }, { session });
      if (!account) throw new Error('Account not found');

      await accounts.updateOne(
        { account_id: parseInt(accountId, 10) },
        {
          $set: { limit: 12000, last_transaction_date: currentDate },
          $inc: { transaction_count: 1 }
        },
        { session }
      );

      await customers.updateMany(
        { accounts: { $in: [parseInt(accountId, 10)] } },
        {
          $set: { last_transaction_date: currentDate },
          $inc: { transaction_count: 1 }
        },
        { session }
      );

      await transactions.insertOne(
        {
          account_id: parseInt(accountId, 10),
          transaction_count: (account.transaction_count || 0) + 1,
          bucket_start_date: currentDate,
          bucket_end_date: currentDate,
          transactions: [{ date: currentDate, amount: 1500, transaction_code: 'buy' }]
        },
        { session }
      );
    }, transactionOptions);
  } finally {
    await session.endSession();
    await client.close();
  }
}
```

### Python callback transaction example

```py
from pymongo import MongoClient, ReadPreference, WriteConcern
from pymongo.read_concern import ReadConcern
from datetime import datetime

uri = 'your_mongodb_connection_string'

def callback(session, account_id):
    db = session.client.sample_analytics
    accounts = db.accounts
    customers = db.customers
    transactions = db.transactions
    current_date = datetime.utcnow()

    account = accounts.find_one({'account_id': int(account_id)}, session=session)
    if not account:
        raise Exception('Account not found')

    accounts.update_one(
        {'account_id': int(account_id)},
        {'$set': {'limit': 12000, 'last_transaction_date': current_date},
         '$inc': {'transaction_count': 1}},
        session=session)

    customers.update_many(
        {'accounts': {'$in': [int(account_id)]}},
        {'$set': {'last_transaction_date': current_date},
         '$inc': {'transaction_count': 1}},
        session=session)

    transactions.insert_one(
        {'account_id': int(account_id),
         'transaction_count': account.get('transaction_count', 0) + 1,
         'bucket_start_date': current_date,
         'bucket_end_date': current_date,
         'transactions': [{
             'date': current_date,
             'amount': 1500,
             'transaction_code': 'buy'
         }]},
        session=session)


def run(account_id):
    client = MongoClient(uri)
    with client.start_session() as session:
        session.with_transaction(
            lambda s: callback(s, account_id),
            read_concern=ReadConcern('snapshot'),
            write_concern=WriteConcern('majority'),
            read_preference=ReadPreference.PRIMARY)
```

### Ruby callback transaction example

```ruby
require 'mongo'
require 'date'

uri = 'your_mongodb_connection_string'

def transaction_callback(session, account_id)
  db = session.client.use('sample_analytics')
  accounts = db[:accounts]
  customers = db[:customers]
  transactions = db[:transactions]
  current_date = DateTime.now

  account = accounts.find({ 'account_id' => account_id.to_i }, session: session).first
  raise 'Account not found' if account.nil?

  accounts.update_one(
    { 'account_id' => account_id.to_i },
    { '$set' => { 'limit' => 12000, 'last_transaction_date' => current_date },
      '$inc' => { 'transaction_count' => 1 } },
    session: session)

  customers.update_many(
    { 'accounts' => { '$in' => [account_id.to_i] } },
    { '$set' => { 'last_transaction_date' => current_date },
      '$inc' => { 'transaction_count' => 1 } },
    session: session)

  transactions.insert_one(
    { 'account_id' => account_id.to_i,
      'transaction_count' => account['transaction_count'].to_i + 1,
      'bucket_start_date' => current_date,
      'bucket_end_date' => current_date,
      'transactions' => [{
        'date' => current_date,
        'amount' => 1500,
        'transaction_code' => 'buy'
      }]},
    session: session)
end

client = Mongo::Client.new(uri, write_concern: { w: :majority })
client.start_session.with_transaction(
  read_concern: { level: :snapshot },
  write_concern: { w: :majority },
  read: { mode: :primary }
) do |session|
  transaction_callback(session, ARGV[0])
end
```

## 6. Tips and tricks

- Use `withTransaction()` in driver code unless you need a low-level custom flow.
- Keep session lifetime short and always call `endSession()`.
- Use `snapshot` read concern only inside transactions; do not use it for ad hoc reads unless required.
- Avoid transactions that do large array writes or document rewrites.
- If you need to create or modify indexes in a transaction, prefer the `collMod` pattern or do it outside the transaction in production.
- Retry commit failures only on `UnknownTransactionCommitResult`.
- When a transaction aborts, inspect the `errorLabels` array.
- Model data to minimize cross-collection transactions.

## 7. Answering style

- Always mention that MongoDB supports ACID only when using replica sets or sharded clusters with WiredTiger.
- Explain the difference between single-document atomicity and multidocument transactions.
- Give clear driver-specific examples for Node.js, Python, and Ruby.
- Emphasize retry semantics, short-lived sessions, and the costs of long-running transactions.
