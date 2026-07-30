---
name: database-mongodb-lookup
description: Guides the agent on MongoDB $lookup joins, including simple equality joins, pipeline lookups, sharded collection behavior, and output shaping with $mergeObjects.
---

# MongoDB $lookup Joins

You are an expert on MongoDB `$lookup`. When asked about joining collections, explain left outer join semantics, `localField/foreignField`, pipeline lookup patterns, and optimization strategies.

## 1. Simple equality joins

- `$lookup` performs a left outer join from the current collection to another collection.
- The result is an array field containing matching documents.

Example:

```js
{
  $lookup: {
    from: 'customers',
    localField: 'account_id',
    foreignField: 'accounts',
    as: 'customer_details'
  }
}
```

- If no matches exist, `customer_details` is an empty array.
- The joined field can be unwound or merged as needed.

## 2. Pipeline lookups

- Use `pipeline` and `let` for advanced joins.
- This is necessary when joining on computed fields or when filtering the joined collection.

Example:

```js
{
  $lookup: {
    from: 'accounts',
    let: { acctId: '$account_id' },
    pipeline: [
      { $match: { $expr: { $in: ['$$acctId', '$accounts'] } } },
      { $project: { _id: 0, account_type: 1, status: 1 } }
    ],
    as: 'account_details'
  }
}
```

## 3. Flattening the joined result

- Use `$unwind` when the result array must become a single embedded document.
- Use `$mergeObjects` or `$replaceRoot` to collapse joined fields.

Example:

```js
{ $unwind: '$account_details' },
{ $replaceRoot: { newRoot: { $mergeObjects: ['$account_details', '$$ROOT'] } } },
{ $unset: 'account_details' }
```

### Warning

- Use this pattern only when each input document should match one joined document.
- If multiple matches exist, `$unwind` will multiply documents.

## 4. Optimizing $lookup

- Ensure `foreignField` is indexed in the joined collection.
- Filter the left-side documents before `$lookup` with `$match`.
- When possible, use `$limit` or `$project` inside the lookup pipeline to reduce data transfer.
- Avoid joining against very large collections without selective conditions.

## 5. Sharded collection behavior

- MongoDB 5.1+ supports `$lookup` with sharded collections.
- The local and from collections can be sharded.
- The join may be more expensive if the join key is not aligned with chunk distribution.
- Prefer good shard keys and indexed foreign fields.

## 6. Views with $lookup

- Create read-only views that wrap `$lookup` pipelines.
- Use `db.createView('enriched_transactions', 'transactions', pipeline)`.
- Views cannot include `$out` or `$merge`.

Example:

```js
db.createView('enriched_transactions', 'transactions', [
  { $lookup: { from: 'customers', localField: 'account_id', foreignField: 'accounts', as: 'customer_details' } },
  { $set: { customer: { $arrayElemAt: ['$customer_details', 0] } } },
  { $unset: 'customer_details' }
]);
```

## 7. Tips and tricks

- Use `$arrayElemAt` to extract the first join match when you expect one-to-one semantics.
- Use `$lookup` + `$unwind` + `$replaceRoot` to flatten results into a single document.
- Use `$lookup` pipeline to filter out unnecessary fields: select only the projected fields you need.
- When multiple join results are valid, keep the result array or use `$limit` inside the `pipeline`.
- For large join arrays, consider denormalization instead of repeated `$lookup`.

## 8. Answering style

- Explain that `$lookup` is a left outer join, not an inner join.
- Show simple and pipeline-based examples.
- Stress indexing the foreign field.
- Mention sharded collections only when the user asks about production or Atlas.
- Use `routes`, `transactions`, `customers`, and `accounts` examples.
