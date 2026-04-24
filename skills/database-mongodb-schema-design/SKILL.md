---
name: database-mongodb-schema-design
description: Guides the agent on MongoDB schema design, workload-driven modeling, embedding versus referencing, and applying data model rules for flexible document schemas.
---

# MongoDB Schema Design

You are an expert on MongoDB schema design. When asked about modeling documents in MongoDB, explain how to derive a schema from workload, map relationships, choose embedding versus references, and optimize for read/write patterns.

## 1. Design from workload, not from tables

- Start by identifying the most important application queries and write operations.
- Capture: action, query type, fields involved, frequency, and priority.
- Use the workload to guide whether data should be embedded, referenced, or denormalized.
- For an airline route system, prioritize high-volume read patterns such as searching routes by `src_airport`/`dst_airport`, checking route details, and infrequent writes to route metadata.

### Example workload table

| Action | Type | Fields | Frequency | Priority |
|---|---|---|---|---|
| Add new route | Write | `flight_id`, `airline`, `src_airport`, `dst_airport`, `airplane`, `stops`, `codeshare` | 150/day | High |
| Search routes | Read | `src_airport`, `dst_airport` | 22,000/day | High |
| Update route details | Write | `flight_id`, `src_airport`, `dst_airport`, `airline` | 100/day | Medium |
| Check route info | Read | `src_airport`, `dst_airport`, `airplane`, `stops` | 40,000/day | High |

## 2. Relationship mapping

- One-to-one: one document contains one other document or subdocument.
- One-to-many: one parent document contains an array of child values or references.
- Many-to-many: use arrays of references or separate linking collections.

### Airline route example

- Airline-to-routes: one-to-many.
- Airports-to-routes: many-to-many.
- Use embedding for small, frequently-read reference data.
- Use references when the referenced document is large, frequently updated, or shared across many documents.

## 3. Embedding vs referencing

### Embedding

- Best when related data is read together and changes infrequently.
- Avoid `$lookup` and multiple round trips.
- Use for airline details inside route documents.

Example:

```js
{
  _id: ObjectId('56e9b39b732b6122f877facc'),
  flight_id: 'FL123',
  airline: { id: 410, name: 'Delta Airlines', alias: '2B', iata: 'ARD' },
  src_airport: 'JFK',
  dst_airport: 'LAX',
  codeshare: '',
  stops: 0,
  airplane: 'ATP'
}
```

### Referencing

- Best when data is large, shared, or updated independently.
- Use for airports or other heavy objects that are reused across many routes.

Example airport document:

```js
{
  _id: 'JFK',
  name: 'JFK International Airport',
  location: { city: 'New York', country: 'USA' },
  facilities: ['Wi-Fi', 'Lounge', 'VIP Services']
}
```

### Hybrid pattern: subset embedding

- Embed only frequently-used airport fields inside the route document.
- Keep full airport metadata in a separate collection.

Example subset:

```js
{
  _id: ObjectId('56e9b39b732b6122f877fa35'),
  flight_id: 'FL123',
  airline: { id: 410, name: 'Delta Airlines', alias: '2B', iata: 'ARD' },
  src_airport: { code: 'JFK', name: 'JFK International Airport' },
  dst_airport: { code: 'LAX', name: 'Los Angeles International Airport' },
  airplane: 'CR2',
  stops: 0
}
```

## 4. Modeling principles

- Model for the most common access pattern first.
- Duplicate small, read-heavy values if it avoids extra lookups.
- Keep documents under 16 MB.
- Avoid embedding large unrelated subdocuments.
- Prefer a single collection when queries naturally target one document shape.

### Relational vs document mindset

- Do not model MongoDB like normalized SQL unless the access patterns justify it.
- Group fields by how the application reads them, not by how they are logically connected.
- Expect different documents in the same collection to vary in shape.

## 5. Schema changes and evolution

- Flexible schema means you can add new fields over time, but you still need a plan.
- Use a schema version field when the document shape changes significantly.

Example:

```js
{
  _id: 'ABC123',
  schema_version: 2,
  flight_id: 'FL123',
  airline: { id: 410, name: 'Delta Airlines' },
  src_airport: 'JFK',
  dst_airport: 'LAX',
  airplane: 'ATP',
  new_field: 'value'
}
```

## 6. Tips and tricks

- Use `ObjectId` or natural key `_id` intentionally; do not mix them randomly.
- If a field is read frequently but updated rarely, embed it.
- If a field is updated frequently and shared across documents, reference it.
- Use arrays for bounded child collections; avoid unbounded arrays that grow without limit.
- Sketch query paths first, then turn them into a schema diagram.
- Create sample documents for each major query path and verify they are fetched in one round trip.

## 7. Answering style

- When asked about schema design, lead with workload and query patterns.
- Provide the route/airport example and mention embedding airline data directly in routes.
- Mention tradeoffs explicitly: faster reads versus harder writes.
- Call out when a document should be split into a separate collection.
- Avoid vague statements; prefer concrete rules and code snippets.
