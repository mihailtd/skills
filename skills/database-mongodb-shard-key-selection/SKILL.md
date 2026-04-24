---
name: database-mongodb-shard-key-selection
description: Guides the agent on selecting and evaluating MongoDB shard keys, including cardinality, access patterns, and range vs hashed keys.
---

# MongoDB Shard Key Selection

You are an expert on MongoDB shard key selection. When asked about choosing a shard key, explain the criteria, the tradeoffs of range and hashed keys, and how to validate a candidate before sharding.

## 1. Shard key criteria

A good shard key should:
- have high cardinality so chunks can split into many small ranges
- have an even distribution of values
- appear in the most important query filters to enable targeted routing
- avoid monotonically increasing patterns when using range sharding
- be stable and not change frequently

## 2. Cardinality and distribution

- High cardinality means many unique values, reducing the chance that one key value dominates a chunk.
- Even distribution means no single value is over-represented.
- Low-cardinality keys like boolean or country codes often create hot shards or jumbo chunks.
- Examine `db.collection.getShardDistribution()` and metrics after sharding to validate distribution.

### Example

```js
db.orders.createIndex({ customerId: 1, orderDate: 1 });
sh.shardCollection('sales.orders', { customerId: 1 });
```

If `customerId` is not selective enough, prefer a compound shard key such as `{ customerId: 1, orderDate: 1 }`.

## 3. Query alignment

- If your queries include the shard key, the query can be routed to a single shard.
- If the query lacks the shard key, MongoDB may scatter the query to all shards and perform a `SHARD_MERGE`.
- For shard-key-prefixed compound indexes, queries can still target a single shard when the prefix fields are present.

### Example

A query filtered by `{ country: 'US', city: 'Seattle' }` is more targetable if the shard key is `{ country: 1, city: 1 }`.

## 4. Range sharding vs hashed sharding

### Range sharding
- Maintains order of shard key values.
- Efficient for range scans and sorted queries on the shard key.
- Vulnerable to insert hotspots for monotonically increasing keys like timestamps or sequential IDs.
- Example:

```js
sh.shardCollection('sales.orders', { orderDate: 1 });
```

### Hashed sharding
- Applies a hash to the key before distribution.
- Spreads writes evenly across shards.
- Breaks range queries and makes sort operations on the original key inefficient.
- Only one field can be hashed.
- Example:

```js
sh.shardCollection('sales.orders', { orderDate: 'hashed' });
```

## 5. When to use hashed shard keys

- Use hashed sharding when the best shard key field is monotonically increasing.
- Use hashed sharding when even write distribution is more important than range query performance.
- Avoid hashed keys when the workload depends on shard-key range scans or sorted retrievals.

## 6. Using analyzeShardKey

- In MongoDB 7.0+, `analyzeShardKey()` helps validate a candidate key.
- It reports query targeting, frequency, and distribution.

### Example

```js
db.routes.configureQueryAnalyzer({ mode: 'full', samplesPerSecond: 1 });
const analysis = db.routes.analyzeShardKey({ src_airport: 1, dst_airport: 1 });
printjson(analysis);
```

- Look for high `targeted` query percentages and low `chunksPerShard` skew.
- If `collisions` or `frequency` show uneven query patterns, choose a different key.

## 7. Compound shard keys and prefix rules

- Compound shard keys are often the best compromise between distribution and query targeting.
- The shard key must be the prefix of any supporting index used for query routing.
- Example:

```js
sh.shardCollection('sales.orders', { customerId: 1, orderDate: 1 });
```

- Query filters like `{ customerId: 123, orderDate: { $gte: ISODate('2025-01-01') } }` can target a single shard.

## 8. Avoiding poor shard keys

Bad shard key patterns:
- monotonic values with range sharding, e.g. `{ _id: 1 }` or `{ createdAt: 1 }`
- low-cardinality fields, e.g. `{ country: 1 }`
- fields updated frequently or changed after insertion
- fields absent from the most common query filters

## 9. Answering style

- Make the user feel the shard key choice is the most important decision in the sharding project.
- Use the phrase “high cardinality,” “query targeting,” and “non-monotonic range” frequently.
- Provide both range and hashed examples, and explain their tradeoffs clearly.
- Mention `analyzeShardKey()` as the modern way to validate a candidate.
- Advise against sharding until a good shard key candidate exists.
