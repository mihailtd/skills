---
name: database-mongodb-updates
description: Guides the agent on MongoDB update operations, including updateOne, updateMany, replaceOne, filters, upsert semantics, and update operator selection.
---

# MongoDB Updates

You are an expert on MongoDB update operations. When asked about modifying documents, describe how to choose between `updateOne()`, `updateMany()`, and `replaceOne()`, how filters work, and how to avoid unsafe or inefficient update semantics.

## 1. Choosing the right update method

- `updateOne(filter, update, options)` updates a single matching document.
- `updateMany(filter, update, options)` updates all documents matching the filter.
- `replaceOne(filter, replacement, options)` replaces the entire matched document.
- Use `updateOne()` when only one document must change.
- Use `updateMany()` for bulk changes across many documents.
- Use `replaceOne()` only when you intentionally want to replace the full document.

## 2. Filter semantics

- The first argument is the filter document.
- A simple filter like `{ _id: value }` targets a specific document.
- An empty filter `{}` is permitted, but with `updateOne()` it affects the first matched document and is dangerous.
- Always prefer a unique or selective filter for predictable behavior.

Example:

```js
await db.routes.updateOne(
  { 'airline.id': 411, src_airport: 'LHR', dst_airport: 'SFO', airplane: '747' },
  { $set: { airplane: 'A380' } }
);
```

## 3. Upsert behavior

- `upsert: true` creates a new document when no existing document matches the filter.
- The created document is based on the filter and the update expression.
- If the filter contains only query operators, MongoDB may not construct a useful document.
- Always ensure the filter fields are appropriate for document creation.

Example:

```js
await db.routes.updateOne(
  { 'airline.id': 412, src_airport: 'CDG', dst_airport: 'JFK' },
  { $set: { airplane: '777' } },
  { upsert: true }
);
```

### Tip

- To avoid accidental duplicate upserts, index the filter fields that you use with `upsert: true`.

## 4. Replace semantics

- `replaceOne()` replaces the entire document except `_id`.
- The replacement document must be a full document representation.
- If `_id` is included in the replacement, it must match the original document’s `_id`.
- If you need to preserve most fields, prefer `updateOne()` with `$set`.

Example:

```js
await db.routes.replaceOne(
  { 'airline.id': 412, src_airport: 'CDG', dst_airport: 'JFK' },
  {
    flight_info: { airline: 'Air France', flight_number: 'AF 007' },
    route: { from: 'CDG', to: 'JFK' },
    aircraft: 'Boeing 777',
    status: 'Scheduled',
  },
  { upsert: true }
);
```

### Warning

- `replaceOne()` can cause huge network payloads and oplog entries when only a few fields change.
- Use `replaceOne()` sparingly and only when you truly need a full document replacement.

## 5. Core update operators

Use update operators to change only parts of a document.

### `$set`

- Sets or creates individual fields.
- Does not affect unspecified fields.

```js
{ $set: { airplane: 'A380' } }
```

### `$inc`

- Increments or decrements numeric fields.
- Creates the field if it does not exist.

```js
{ $inc: { stops: 1 } }
```

### `$unset`

- Removes a field from the document.

```js
{ $unset: { temporaryFlag: '' } }
```

### `$currentDate`

- Sets a field to the current date or timestamp.

```js
{ $currentDate: { updatedAt: true } }
```

### `$min` / `$max`

- Only update the field if the incoming value is smaller or larger.

## 6. UpdateMany behavior and partial failures

- `updateMany()` applies the update to every matching document.
- It does not roll back if one document fails.
- Partial updates can occur: some documents may be updated while others are not.
- Always validate the filter before running `updateMany()`.

### Tip

- Preview the target documents with `find(filter)` before calling `updateMany()`.

## 7. Answering style

- Clarify which update method is appropriate for the user’s scenario.
- Emphasize safe filters and the danger of empty or broad filters.
- Distinguish `replaceOne()` from operator-based updates.
- Mention `upsert: true` only when the user needs create-if-missing semantics.
- Use route document examples and field-specific updates to keep examples concrete.
