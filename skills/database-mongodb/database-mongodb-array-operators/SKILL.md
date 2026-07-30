---
name: database-mongodb-array-operators
description: Guides the agent on MongoDB array update operators, including $push, $each, $sort, $slice, $addToSet, $pull, $pop, positional updates, and arrayFilters.
---

# MongoDB Array Update Operators

You are an expert on MongoDB array operations. When asked about arrays inside documents, explain how to append, remove, update, and filter array elements safely and efficiently.

## 1. Appending values with $push

- Use `$push` to add elements to an array.
- If the array does not exist, MongoDB creates it.

Example:

```js
await db.routes.updateOne(
  { 'airline.id': 413, src_airport: 'DFW', dst_airport: 'LAX' },
  { $push: { prices: { class: 'business', price: 2500 } } }
);
```

### `$each`, `$sort`, and `$slice`

- `$each` appends multiple values at once.
- `$sort` sorts the array after appending.
- `$slice` trims the array to a fixed length.

Example:

```js
await db.routes.updateOne(
  { 'airline.id': 413, src_airport: 'DFW', dst_airport: 'LAX' },
  {
    $push: {
      prices: {
        $each: [
          { class: 'economy', price: 800 },
          { class: 'first', price: 2000 },
        ],
        $sort: { price: 1 },
        $slice: -3,
      },
    },
  }
);
```

### Tip

- Use `$slice` to maintain a top-N or fixed-size array.
- Use `$sort` on a field when the array contains objects and you need deterministic order.

## 2. Preventing duplicates with $addToSet

- Use `$addToSet` when you want unique array entries.
- It adds the value only if it does not already exist exactly.

Example:

```js
await db.routes.updateOne(
  { 'airline.id': 413, src_airport: 'DFW', dst_airport: 'LAX' },
  { $addToSet: { prices: { class: 'economy plus', price: 1200 } } }
);
```

### Tip

- `$addToSet` checks complete document equality for objects, not only a subset of fields.
- Use this for arrays of scalars or objects when duplicates must be avoided.

## 3. Removing values with $pull and $pop

### `$pull`

- Removes all array elements that match the query condition.

```js
await db.routes.updateOne(
  { 'airline.id': 413, src_airport: 'DFW', dst_airport: 'LAX' },
  { $pull: { prices: { class: 'first', price: 2000 } } }
);
```

### `$pop`

- Removes the first element with `$pop: -1` or the last element with `$pop: 1`.

```js
await db.routes.updateOne(
  { 'airline.id': 413, src_airport: 'DFW', dst_airport: 'LAX' },
  { $pop: { prices: 1 } }
);
```

## 4. Updating specific array elements

### Direct index updates

- Use positional index notation when you know the element position.

```js
await db.routes.updateOne(
  { 'airline.id': 413, src_airport: 'DFW', dst_airport: 'LAX' },
  { $set: { 'prices.1.price': 3500 } }
);
```

### Positional operator `$`

- Use `$` to update the first array element that matches a query condition.

```js
await db.routes.updateOne(
  {
    'airline.id': 413,
    src_airport: 'DFW',
    dst_airport: 'LAX',
    'prices.class': 'luxury',
  },
  { $set: { 'prices.$.price': 4500 } }
);
```

### Tip

- The positional operator updates the first matching array element in the query.
- If multiple elements match, only the first is updated.

## 5. Filtered positional updates with arrayFilters

- Use `$[<identifier>]` together with `arrayFilters` for fine-grained updates.
- This is the best choice when you need to update multiple matching array elements conditionally.

Example:

```js
await db.routes.updateOne(
  { 'airline.id': 413, src_airport: 'DFW', dst_airport: 'LAX' },
  { $set: { 'prices.$[elem].price': 2600 } },
  { arrayFilters: [{ 'elem.class': 'business' }] }
);
```

### Placeholders

- `$[]` updates all matching elements.
- `$[<identifier>]` updates elements that satisfy the array filter condition.

### Tip

- `arrayFilters` are evaluated per element and can target nested array object fields.
- Use descriptive identifiers like `elem` or `priceEntry`.

## 6. Answering style

- Focus on the array operator needed by the user: append, remove, update, or prevent duplicates.
- Use concrete route/price document examples.
- Highlight whether the array field exists or should be created.
- Warn about object equality in `$addToSet` and the difference between direct index updates and positional updates.
- Prefer operator-based updates over replacing the full document when only array elements change.
