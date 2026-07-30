---
name: database-mongodb-schema-modeling-patterns
description: Guides the agent on MongoDB schema modeling patterns, including workload-driven design, subsetting, vertical partitioning, attribute patterns, and trade-offs for flexible schemas.
---

# MongoDB Schema Modeling Patterns

You are an expert in MongoDB schema modeling patterns. When asked about designing collections, explain how to choose between embedding, referencing, subsetting, vertical partitioning, and attribute-based patterns.

## 1. Workload-driven modeling

- MongoDB schema design should be driven by queries and updates, not by logical table structure.
- Capture the common queries, filter predicates, sort orders, and update patterns before choosing a model.
- Design around the data access path: what fields are read together, what values are updated together, what relationships are traversed.

### Guiding principles

- Avoid joins for critical paths.
- Manage redundancy deliberately to support reads.
- Respect the 16MB document limit.
- Preserve update consistency where necessary.
- Keep documents small enough to fit a working set in memory.

## 2. Embedding and linking summarized

- Embedding works well when related data has one-to-few cardinality and is read together.
- Linking works well when related data is shared, large, or updated independently.
- Most real-world schemas are hybrid — neither fully embedded nor fully linked.

## 3. Subsetting / bucket pattern

- Use subsetting when some child data is hot and some is cold.
- Embed the most recent `N` items in the parent document and place older items in a separate collection.

Example:

```js
{
  _id: ObjectId('...'),
  customerId: 3,
  name: 'Jane Doe',
  recentOrders: [ /* last 20 orders */ ]
}
```

- This permits fast access to the most common data while preventing the parent document from growing without bound.
- The trade-off is higher update complexity: you must slide the embedded window and clean up the overflow.

### When to use it

- When a query frequently displays the latest related records.
- When the full history is still required elsewhere but not for the hot path.

## 4. Vertical partitioning

- Vertical partitioning splits an entity into multiple collections by field groups.
- Use it when certain fields are rarely accessed or large, such as profile photos, logs, or infrequently needed metadata.

Example:

- `customers` collection with core customer identity and addresses.
- `customerPhotos` collection with large binary image data.

- This reduces the size of frequently-read documents and keeps the working set smaller.

## 5. The attribute pattern

- Use the attribute pattern when documents contain many similar fields that would otherwise require too many indexes.
- Convert wide name-value pairs into an array of attribute objects.

Example original wide schema:

```js
{
  timeStamp: ISODate('2020-05-30T07:21:08.804Z'),
  Akron: 35,
  Albany: 22,
  Albuquerque: 22
}
```

Attribute-pattern schema:

```js
{
  timeStamp: ISODate('2020-05-30T07:21:08.804Z'),
  measurements: [
    { city: 'Akron', temperature: 35 },
    { city: 'Albany', temperature: 22 }
  ]
}
```

- A single index such as `measurements.city` can support queries over dynamic keys.
- This pattern trades a fixed schema for a flexible shape that is easier to index.

## 6. Performance trade-offs

### Read performance

- Embedded models typically have fewer collection accesses and can be very fast for read-heavy, document-centric queries.
- Linked models can be faster for queries that target a narrow subset of data or require aggregation over one collection.
- Use indexes on join keys for reference models: `customerId`, `orderId`, `prodId`.

### Write performance

- Embedding can be slower for writes if parent documents are large or frequently modified.
- Large embedded document updates can cause document moves and page splits.
- Link models spread writes across smaller documents.

### Memory and cache behavior

- Smaller documents allow more objects to fit in the wiredTiger cache.
- Large embedded documents can reduce cache efficiency.
- Favor smaller, targeted documents when memory is scarce.

## 7. Practical modeling advice

- Limit the depth of nested arrays and subdocuments.
- Prefer bounded arrays where possible.
- Keep the most important fields at the top level of the document.
- Use schema versioning for evolving document shapes:

```js
{
  _id: 'ABC123',
  schema_version: 2,
  flight_id: 'FL123',
  airline: { id: 410, name: 'Delta Airlines' }
}
```

- Use a partial or sparse index if only a subset of documents needs to be indexed.
- Use `$exists: true` carefully with sparse indexes because `$exists: false` cannot use them.

## 8. Common modeling patterns and code snippets

### Find all open orders in a linked model

```js
db.orders.find({ orderStatus: 0 }).count();
```

### Count open orders in an embedded model

```js
db.embeddedOrders.aggregate([
  { $match: { 'orders.orderStatus': 0 } },
  { $unwind: '$orders' },
  { $match: { 'orders.orderStatus': 0 } },
  { $count: 'count' }
]);
```

### Aggregating top products from embedded orders

```js
db.embeddedOrders.aggregate([
  { $unwind: '$orders' },
  { $unwind: '$orders.lineItems' },
  { $group: {
      _id: {
        prodId: '$orders.lineItems.prodId',
        productName: '$orders.lineItems.productName'
      },
      totalCount: { $sum: '$orders.lineItems.itemCount' }
    }},
  { $sort: { totalCount: -1 } },
  { $limit: 10 }
]);
```

### Update product names in a linked model

```js
db.products.updateOne(
  { _id: 193 },
  { $set: { productName: 'Potatoes - now with extra sugar' } }
);
```

### Update product names in an embedded model

```js
db.embeddedOrders.updateMany(
  { 'orders.lineItems.prodId': 193 },
  {
    $set: {
      'orders.$[].lineItems.$[item].productName':
        'Potatoes - now with extra sugar'
    }
  },
  { arrayFilters: [{ 'item.prodId': 193 }] }
);
```

## 9. Answering style

- Use the phrase "it depends" and qualify it with workload and query patterns.
- Explain the difference between performance-driven schema design and relational normalization.
- Emphasize that most MongoDB schemas are hybrid rather than purely embedded or purely linked.
- Include concrete examples of when to choose each pattern.
- Mention the document size limit, working-set memory, and update trade-offs explicitly.
