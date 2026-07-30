---
name: database-mongodb-aggregation-join-optimization
description: Guides the agent on MongoDB aggregation join optimization, including $lookup indexing, join order, $unwind fusion, and $graphLookup tuning.
---

# MongoDB Aggregation Join Optimization

You are an expert on optimizing MongoDB aggregation joins. When asked about joins in pipelines, explain how to index foreign keys, choose join order, and minimize the cost of $lookup and $graphLookup.

## 1. $lookup basics

- `$lookup` performs a left outer join from the current pipeline documents into another collection.
- It executes once for each document entering the join stage.
- Performance depends on the number of input documents and the efficiency of the foreign collection lookup.

### Example

```js
db.customers.aggregate([
  { $lookup: {
      from: 'orders',
      localField: '_id',
      foreignField: 'customerId',
      as: 'orders'
    }
  }
]);
```

## 2. Index the foreign field

- Always index the `foreignField` used in `$lookup`.
- Without an index, each lookup may require a collection scan over the joined collection.

### Example

```js
db.orders.createIndex({ customerId: 1 });
```

### Tip

- If the `foreignField` is nested, index the exact dotted path.
- If the lookup uses a compound foreign field, ensure the index matches the lookup key pattern.

## 3. Join order matters

- Choose the smallest effective input set for `$lookup`.
- If filtering on the local collection is possible, apply `$match` first.
- If either side of the join can be prefiltered, do so before the join.

### Example: reduce join input

```js
db.lineitems.aggregate([
  { $match: { status: 'shipped' } },
  { $lookup: {
      from: 'products',
      localField: 'prodId',
      foreignField: '_id',
      as: 'product'
    }
  }
]);
```

- This is better than joining all line items and then filtering.

## 4. Join small-to-large when possible

- If both join orders are feasible, join from the smaller source collection to the larger one.
- This reduces the number of lookup operations.

### Example

- `db.customers.aggregate([ $lookup: { from: 'orders', ... } ])` is usually better than
- `db.orders.aggregate([ $lookup: { from: 'customers', ... } ])` when orders outnumber customers.

### Tip

- Join the smaller side first when the join condition is symmetrical and both sides are indexed.

## 5. `$unwind` fusion

- If you immediately unwind a `$lookup` result, MongoDB may fuse the `$lookup` and `$unwind` into a single stage.
- This avoids building large joined arrays that are then immediately flattened.

### Example

```js
db.users.aggregate([
  { $lookup: {
      from: 'comments',
      localField: 'email',
      foreignField: 'email',
      as: 'comments'
    }
  },
  { $unwind: '$comments' }
]);
```

- The engine can optimize this into a single fused stage.

## 6. Reducing join volume with `$limit` and `$sort`

- If your pipeline only needs top N joined results, apply `$sort` and `$limit` before `$lookup` when it preserves semantics.
- This avoids joining large intermediate result sets unnecessarily.

### Example

```js
db.lineitems.aggregate([
  { $group: { _id: { orderId: '$orderId', prodId: '$prodId' }, itemCount: { $sum: '$itemCount' } } },
  { $sort: { itemCount: -1 } },
  { $limit: 5 },
  { $lookup: { from: 'orders', localField: '_id.orderId', foreignField: '_id', as: 'orders' } }
]);
```

- This is far more efficient than joining all groups and then limiting.

## 7. `$graphLookup` optimization

- `$graphLookup` performs recursive traversal across relationships.
- Index `connectToField` to make each traversal step efficient.
- Limit depth with `maxDepth` and use `restrictSearchWithMatch` when possible.

### Example

```js
db.socialGraph.aggregate([
  { $match: { person: 1476767 } },
  { $graphLookup: {
      from: 'socialGraph',
      startWith: [1476767],
      connectFromField: 'friends',
      connectToField: 'person',
      maxDepth: 2,
      depthField: 'Depth',
      as: 'GraphOutput'
    }
  },
  { $unwind: '$GraphOutput' }
], { allowDiskUse: true });
```

### Tip

- Graph traversals expand exponentially; keep `maxDepth` as low as possible.
- A `connectToField` index is usually the difference between scalable graph traversal and a slow query.

## 8. Join performance diagnostics

- Aggregation explain output may not show whether `$lookup` uses an index internally.
- Look at overall stage timing and compare indexed vs non-indexed joins empirically.
- If `$lookup` is slow, ensure the foreign collection has an appropriate index and the input set is small.

### Practical check

- Create a join index and re-run `explain()`.
- Compare stage ms timings before and after adding the index.
- If the join time drops dramatically, the index was the missing piece.

## 9. Tips and tricks

- Index the exact `foreignField` used in `$lookup`.
- Build compound indexes when `$lookup` involves multiple join fields.
- Prefer filtering before `$lookup` rather than after.
- For large joins, consider streaming results or a materialized summary instead of full embedded arrays.
- Avoid `$lookup` in high-frequency paths unless the join input is already very small.

## 10. Answering style

- Emphasize that join performance is driven by input size and lookup efficiency.
- Use terms like `$lookup`, `foreignField`, `lookup order`, `$unwind`, and `$graphLookup`.
- Explain why small-to-large join order and early filtering matter.
- Give concrete examples of index creation and pipeline order changes.
