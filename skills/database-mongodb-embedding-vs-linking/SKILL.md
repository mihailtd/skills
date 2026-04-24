---
name: database-mongodb-embedding-vs-linking
description: Guides the agent on MongoDB embedding and linking strategies, including hybrid patterns, array constraints, update costs, and when to choose one approach over another.
---

# MongoDB Embedding vs Linking

You are an expert on MongoDB document modeling trade-offs. When asked about embedding, referencing, or hybrid models, explain the performance, consistency, size, and update cost implications in detail.

## 1. Core decision: embed or link?

### Embedding

- Embed when related data is usually read together.
- Embedding avoids joins and additional collections.
- Best for data with a one-to-few relationship and bounded array sizes.
- Example: embedding order line items inside a customer or order document when the item count is small and order details are accessed together.

Example embedded order document:

```js
{
  _id: 1,
  email: 'rriggert0@geocities.com',
  first_name: 'Rolando',
  last_name: 'Riggert',
  orders: [
    {
      orderId: 492,
      orderDate: ISODate('2017-08-20T11:51:04.934Z'),
      orderStatus: 6,
      lineItems: [
        {
          prodId: 115,
          productName: 'Juice - Orange',
          price: 4.93,
          itemCount: 172
        }
      ]
    }
  ]
}
```

- Embedding is especially strong for lookup workflows such as "get all data for a customer".
- It also supports atomic updates on the embedded document with a single write.

### Linking (referencing)

- Link when data is large, shared, or updated independently.
- Use references to separate collections for many-to-many relationships or unbounded child collections.
- Reference patterns are closer to normalized SQL models but still should be optimized for MongoDB access patterns.

Example linked collections for orders:

```js
// customers
{ _id: 3, first_name: 'Danyette', last_name: 'Flahy', email: 'dflahy2@networksolutions.com' }

// orders
{ _id: 1, orderDate: ISODate('2017-03-09T16:30:16.415Z'), orderStatus: 0, customerId: 3 }

// lineitems
{ _id: ObjectId(...), orderId: 1, prodId: 158, itemCount: 48 }
```

- With references, queries often require aggregation or application-side joins.
- Use `$lookup` only when joins are an exception, not the rule.

## 2. Hybrid and subsetting patterns

- Hybrid models combine embedding and linking to balance read and write costs.
- Subsetting is a common hybrid pattern: store the most recent or most important child documents embedded, and keep the rest in a separate collection.
- Example: embed the most recent 20 orders in the customer document, while older orders remain in an `orders` collection.

Pseudo-code for a hybrid update:

```js
let customer = db.hybridCustomers.findOne({ _id: customerId });
let orders = customer.orders;
orders.unshift(newOrder);
if (orders.length > 20) orders.pop();
db.hybridCustomers.updateOne({ _id: customerId }, { $set: { orders } });
```

- The hybrid approach improves common lookup performance but adds update complexity and array reordering cost.

## 3. Document size and memory considerations

- MongoDB documents have a 16 MB size limit.
- Large documents reduce the number of documents that fit in memory, increasing IO.
- Avoid embedding large or unbounded arrays that may grow without limit.
- Use bounded arrays or subdocuments only when the size can be controlled.

### Practical rule

- If a child array can grow beyond a few hundred items, consider linking or subsetting.
- If embedded values are updated frequently across many parent documents, references are usually better.

## 4. Consistency and atomicity

- Embedding supports atomic updates within the same document.
- Referenced data may require transactions for multi-collection updates.
- If strong consistency is required for related fields and the write set is small, embedding can simplify the logic.
- If the update set is large or widely shared, references reduce the blast radius.

## 5. Update and write cost trade-offs

### Embedded write cost

- `updateOne({ _id }, { $push: { orders: orderData } })` can be expensive if the document is large or requires moving data on disk.
- If the parent document lacks free padding space, MongoDB may perform document moves or page splits.

### Referenced write cost

- Inserts into multiple collections are more numerous but individually simpler.
- Example:

```js
let rc1 = db.orders.insertOne(orderData);
let rc2 = db.lineitems.insertMany(lineItemsArray);
```

- Referenced writes can outperform embedded writes when the embedded document is large and updates are frequent.

## 6. Example performance observations

- Fetching all customer data is usually fastest with embedding.
- Querying open orders is usually fastest with a dedicated `orders` collection.
- Aggregating top products from embedded orders may require `unwind` and expensive scanning, while the linked model can aggregate more efficiently on a narrower collection.
- Updating repeated product fields in embedded line items can be dramatically slower than updating a single product document in a referenced collection.

## 7. Best practice tips

- Sketch the real application queries first and only then choose embedding or linking.
- If you can answer the common query in one document read, embedding is a strong candidate.
- If the child collection is unbounded, use references or hybrid subsetting.
- Add indexes to lookup keys used by joins: `customerId`, `orderId`, `prodId`.
- Avoid `$lookup` in high-frequency paths unless the join is small and the foreign key is indexed.
- Use `arrayFilters` carefully when updating embedded array elements, and prefer top-level fields for high-volume writes.
- Treat the choice as a performance trade-off, not a data-modeling dogma.

## 8. Answering style

- Explicitly compare the two extremes: "all embedded" vs "all linked".
- Use concrete examples from orders/customers/products.
- Mention the hybrid subsetting pattern and why it exists.
- Call out the 16MB document limit and memory/cache impact.
- Recommend transaction use only for multi-collection updates when references are used.
