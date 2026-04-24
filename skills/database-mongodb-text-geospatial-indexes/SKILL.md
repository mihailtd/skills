---
name: database-mongodb-text-geospatial-indexes
description: Guides the agent on MongoDB text, wildcard, and geospatial indexes with practical creation patterns, query usage, performance tradeoffs, and limitations.
---

# MongoDB Text, Wildcard, and Geospatial Indexes

You are an expert on MongoDB’s specialized index types. When asked about text search, wildcard indexing, or geospatial queries, explain the engine behavior, creation syntax, performance characteristics, and gotchas.

## 1. Text indexes

### What text indexes do

- Text indexes allow full-text search over string content.
- MongoDB builds one index entry per stemmed term per document.
- Only one text index is allowed per collection.
- Text indexes are always sparse.

### Creating a text index

```js
db.listingsAndReviews.createIndex({ description: 'text' });
```

- You can include multiple fields:

```js
db.listingsAndReviews.createIndex({ summary: 'text', space: 'text' });
```

- You can mix text and regular keys in a compound index:

```js
db.listingsAndReviews.createIndex({ summary: 'text', beds: 1 });
```

### Weights and relevance

- Use `weights` to boost important fields.
- Example:

```js
db.listingsAndReviews.createIndex(
  { summary: 'text', description: 'text' },
  { weights: { summary: 3, description: 2 } }
);
```

### Querying a text index

- Use `$text` with `$search`:

```js
db.listingsAndReviews.find(
  { $text: { $search: 'oven kettle and microwave' } },
  { score: { $meta: 'textScore' }, summary: 1 }
)
.sort({ score: { $meta: 'textScore' } })
.limit(3);
```

- Use `"phrase"` for exact phrase matching and `-term` to exclude terms.

### Performance characteristics

- Each distinct search term triggers a separate text index scan.
- More terms → more scans.
- Exact phrases are not scanned as a single phrase; MongoDB still processes each term.
- Long phrases may be slower than a regular regex or collection scan.

### When text search is the wrong tool

- For exact phrase matching across long text, consider a regex or application-level phrase match.
- If the search query contains many words, expect higher latency.
- If you need multiple text indexes on a collection, MongoDB does not allow it.

### Text index caveats

- `sparse: true` has no effect on text indexes.
- Compound text indexes cannot include multi-key or geospatial fields.
- Only one text index per collection is permitted.
- Sorting by other fields may not be covered by the text index.

## 2. Wildcard indexes

### When to use wildcard indexes

- Wildcard indexes are useful when schema attributes are unpredictable or dynamic.
- They index every field in a subdocument.

### Create a wildcard index

```js
db.mycollection.createIndex({ 'data.$**': 1 });
```

- Wildcard indexes automatically include new fields added later.
- They are especially useful for document shapes with unknown or evolving keys.

### Performance tradeoffs

- Wildcard indexes are similar to single-field indexes for reads.
- They impose high overhead for inserts, updates, and deletes.
- The maintenance cost is often as high as indexing all fields individually.

### When not to use them

- Don’t create wildcard indexes out of laziness.
- If many fields are never queried, the wildcard index adds unnecessary write overhead.
- Prefer targeted indexes when the query workload is known.

## 3. Geospatial indexes

### Geospatial data formats

- MongoDB supports legacy coordinate pairs and GeoJSON.
- Example legacy point:

```js
{
  _id: 1,
  coordinates: [ -79.9081268, 9.3547792 ]
}
```

- Example GeoJSON point:

```js
{
  _id: 1,
  location: {
    type: 'Point',
    coordinates: [ -79.9081268, 9.3547792 ]
  }
}
```

### Index types

- `2dsphere`: use for spherical Earth data / GeoJSON.
- `2d`: use for planar 2D coordinate pairs.

### Create a geospatial index

```js
db.shipwrecks.createIndex({ coordinates: '2dsphere' });
```

- If the field does not contain valid coordinate or GeoJSON data, index creation fails.

### Required indexes for queries

- Most geospatial operators require an index.
- `$near`, `$nearSphere`, and `$geoNear` require a `2dsphere` or `2d` index.
- `$geoWithin` can work without an index, but performance improves with one.

### Example proximity query

```js
db.shipwrecks.find({
  coordinates: {
    $near: {
      $geometry: { type: 'Point', coordinates: [ -79.908, 9.354 ] },
      $minDistance: 1000,
      $maxDistance: 10000
    }
  }
}).limit(1);
```

### Limitations

- Geospatial indexes cannot create covered queries; documents must be fetched.
- You cannot shard on a geospatial field.
- If a collection has multiple geospatial indexes, aggregation stages such as `$geoNear` may require an explicit `key`.
- Having at most one `2d` and one `2dsphere` index avoids errors; otherwise, your query may fail without a specified index key.

### Performance tips

- When using `$near`, `$nearSphere`, or `$geoNear`, use `minDistance`/`maxDistance`.
- If you also need sorting by another field, use `$geoWithin` or `$geoNear` rather than relying on the automatic geospatial sort.
- Think carefully before adding a geospatial index: if the field is rarely queried, the maintenance cost may outweigh the benefit.

## 4. Answering style

- Be explicit about the special-case behavior of text, wildcard, and geospatial indexes.
- Recommend `explain()` and `executionStats` for verification.
- Emphasize that text search is not a universal replacement for normal indexed queries.
- Highlight the high write cost of wildcard indexes and the mandatory nature of geospatial indexes for many operators.
- Give concrete creation examples and the exact limitations that can trip up developers.
