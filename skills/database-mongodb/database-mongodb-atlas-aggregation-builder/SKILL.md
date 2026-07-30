---
name: database-mongodb-atlas-aggregation-builder
description: Guides the agent on MongoDB Atlas aggregation pipeline builder usage, exporting pipelines, and translating Atlas UI output into driver code.
---

# MongoDB Atlas Aggregation Pipeline Builder

You are an expert on MongoDB Atlas aggregation builder. When asked about using the Atlas UI, explain how to compose pipelines, preview results, export code, and integrate pipelines into applications.

## 1. What the Atlas builder does

- The Atlas aggregation builder is a visual pipeline editor.
- It helps create, test, and debug aggregation stages.
- It is not primarily an execution environment; it generates pipelines to export.

## 2. Typical workflow

1. Select a collection in Atlas.
2. Open the Aggregation tab.
3. Add stages incrementally.
4. Preview the pipeline results with sample data.
5. Export the pipeline to mongosh, Python, Java, or another language.

## 3. Exporting pipelines

- Use the `Export Pipeline to Language` button.
- Atlas generates driver-ready code.
- This is useful for copy/pasting a tested pipeline into your application.

Example exported JavaScript:

```js
db.routes.aggregate([
  { $match: { airplane: 'CR2' } },
  { $group: { _id: '$src_airport', totalRoutes: { $sum: 1 } } },
  { $sort: { totalRoutes: -1 } },
  { $limit: 5 }
]);
```

Example exported Python:

```py
pipeline = [
    {"$match": {"airplane": "CR2"}},
    {"$group": {"_id": "$src_airport", "totalRoutes": {"$sum": 1}}},
    {"$sort": {"totalRoutes": -1}},
    {"$limit": 5},
]
result = list(db.routes.aggregate(pipeline))
```

## 4. Atlas-specific capabilities

- Atlas supports the `$search` and `$searchMeta` stages for full-text and vector search.
- `$lookup` and `$graphLookup` work in Atlas pipelines.
- The builder can show sample documents after each stage.
- Use the builder to validate the syntax and stage order before exporting.

## 5. Best practices

- Build the pipeline stage by stage.
- Use the preview panel to verify intermediate output.
- Export early and keep the pipeline code in source control.
- Annotate exported pipelines with comments to preserve intent.
- If the builder output includes generated object IDs or sample syntax, adapt it to your driver’s code style.

## 6. Tips and tricks

- Use the builder to discover stage syntax for new operators like `$setWindowFields` or Atlas search stages.
- If a stage fails, the builder often surfaces the error immediately.
- When using `$search`, ensure Atlas search indexes are configured in the collection.
- Use the builder’s `Explain` or `Performance` tools if available.
- Export not only the final pipeline but also sample JSON documents for reference.

## 7. Answering style

- Recommend Atlas builder for pipeline prototyping, not as a production runtime.
- When the user needs code, show both mongosh and driver export snippets.
- Mention Atlas search only for Atlas users.
- Use the builder names and buttons precisely: `Aggregation` tab, `Export Pipeline to Language`, `Preview Stage Results`.
