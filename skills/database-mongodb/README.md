# Database — MongoDB

MongoDB *database* concepts: aggregation pipelines, indexing, sharding, schema design, transactions, and performance tuning — engine-level, not tied to Python.

For projects using MongoDB as their primary datastore. Pair with `python-mongodb` for the Python driver/ODM layer.

## Install

```bash
npx skills add mihailtd/skills/skills/database-mongodb --all
```

Add `-g` to install globally instead of per-project. Use `--skill <name>` (repeatable) instead of `--all` to cherry-pick individual skills from this package, e.g.:

```bash
npx skills add mihailtd/skills/skills/database-mongodb --skill database-mongodb-aggregation-framework
```

## Skills (33)

- **[database-mongodb-aggregation-framework](database-mongodb-aggregation-framework/SKILL.md)** — Guides the agent on MongoDB aggregation framework fundamentals, pipeline structure, stage order, and best practices for building powerful server-side data transformations.
- **[database-mongodb-aggregation-join-optimization](database-mongodb-aggregation-join-optimization/SKILL.md)** — Guides the agent on MongoDB aggregation join optimization, including $lookup indexing, join order, $unwind fusion, and $graphLookup tuning.
- **[database-mongodb-aggregation-memory-and-views](database-mongodb-aggregation-memory-and-views/SKILL.md)** — Guides the agent on MongoDB aggregation memory limits, disk sorts, views, and materialized view patterns.
- **[database-mongodb-aggregation-pipeline-stages](database-mongodb-aggregation-pipeline-stages/SKILL.md)** — Guides the agent on MongoDB aggregation pipeline stages, stage ordering, stage-specific use cases, and how to combine stages for robust pipelines.
- **[database-mongodb-aggregation-pipeline-tuning](database-mongodb-aggregation-pipeline-tuning/SKILL.md)** — Guides the agent on tuning MongoDB aggregation pipelines, including explain output, stage ordering, pipeline optimization, and early filtering techniques.
- **[database-mongodb-array-operators](database-mongodb-array-operators/SKILL.md)** — Guides the agent on MongoDB array update operators, including $push, $each, $sort, $slice, $addToSet, $pull, $pop, positional updates, and arrayFilters.
- **[database-mongodb-atlas-aggregation-builder](database-mongodb-atlas-aggregation-builder/SKILL.md)** — Guides the agent on MongoDB Atlas aggregation pipeline builder usage, exporting pipelines, and translating Atlas UI output into driver code.
- **[database-mongodb-bulk-operations](database-mongodb-bulk-operations/SKILL.md)** — Guides the agent on MongoDB bulk write operations, including bulkWrite, ordered versus unordered execution, batch splitting, and cross-collection writes in MongoDB 8.0.
- **[database-mongodb-delete-optimization](database-mongodb-delete-optimization/SKILL.md)** — Guides the agent on MongoDB delete optimization, logical deletes, index impact, and strategies for handling high-volume delete workloads.
- **[database-mongodb-dml-performance](database-mongodb-dml-performance/SKILL.md)** — Guides the agent on MongoDB data manipulation performance, including insert/update/delete filtering, index overhead, explain() for DML, write concern, and workload-driven tuning.
- **[database-mongodb-embedding-vs-linking](database-mongodb-embedding-vs-linking/SKILL.md)** — Guides the agent on MongoDB embedding and linking strategies, including hybrid patterns, array constraints, update costs, and when to choose one approach over another.
- **[database-mongodb-explain](database-mongodb-explain/SKILL.md)** — Guides the agent on MongoDB explain(), query planning, execution statistics, plan interpretation, and query tuning with explain output.
- **[database-mongodb-indexing](database-mongodb-indexing/SKILL.md)** — Guides the agent on MongoDB indexing fundamentals, B-tree behavior, selectivity, compound and covering indexes, index scans, partial/sparse indexes, and index maintenance tradeoffs.
- **[database-mongodb-inserts](database-mongodb-inserts/SKILL.md)** — Guides the agent on MongoDB insert operations, including insertOne, insertMany, ordered/unordered writes, custom _id values, batch sizing, and error handling.
- **[database-mongodb-lookup](database-mongodb-lookup/SKILL.md)** — Guides the agent on MongoDB $lookup joins, including simple equality joins, pipeline lookups, sharded collection behavior, and output shaping with $mergeObjects.
- **[database-mongodb-network-roundtrips](database-mongodb-network-roundtrips/SKILL.md)** — Guides the agent on minimizing MongoDB network round trips through projections, batching, application logic, and deployment topology.
- **[database-mongodb-query-tuning](database-mongodb-query-tuning/SKILL.md)** — Guides the agent on MongoDB query tuning, including projections, indexes vs scans, hints, sort optimization, and filter strategies for efficient query plans.
- **[database-mongodb-replication-sharding](database-mongodb-replication-sharding/SKILL.md)** — Guides the agent on MongoDB replica sets, oplog replication, change streams, Atlas sharded clusters, shard-key selection, and high-availability operations.
- **[database-mongodb-resharding](database-mongodb-resharding/SKILL.md)** — Guides the agent on MongoDB resharding and shard key refinement, including procedure, limitations, and practical tradeoffs.
- **[database-mongodb-schema-design](database-mongodb-schema-design/SKILL.md)** — Guides the agent on MongoDB schema design, workload-driven modeling, embedding versus referencing, and applying data model rules for flexible document schemas.
- **[database-mongodb-schema-modeling-patterns](database-mongodb-schema-modeling-patterns/SKILL.md)** — Guides the agent on MongoDB schema modeling patterns, including workload-driven design, subsetting, vertical partitioning, attribute patterns, and trade-offs for flexible schemas.
- **[database-mongodb-shard-balancing](database-mongodb-shard-balancing/SKILL.md)** — Guides the agent on MongoDB shard balancing, chunk distribution, balancer windows, and migration tuning.
- **[database-mongodb-shard-key-selection](database-mongodb-shard-key-selection/SKILL.md)** — Guides the agent on selecting and evaluating MongoDB shard keys, including cardinality, access patterns, and range vs hashed keys.
- **[database-mongodb-sharded-query-tuning](database-mongodb-sharded-query-tuning/SKILL.md)** — Guides the agent on tuning queries for MongoDB sharded clusters, including shard targeting, explain plans, sorts, and aggregation behavior.
- **[database-mongodb-sharding-fundamentals](database-mongodb-sharding-fundamentals/SKILL.md)** — Guides the agent on MongoDB sharding fundamentals, including shards, chunks, routers, and the decision when sharding is appropriate.
- **[database-mongodb-text-geospatial-indexes](database-mongodb-text-geospatial-indexes/SKILL.md)** — Guides the agent on MongoDB text, wildcard, and geospatial indexes with practical creation patterns, query usage, performance tradeoffs, and limitations.
- **[database-mongodb-transaction-hotspot-management](database-mongodb-transaction-hotspot-management/SKILL.md)** — Guides the agent on MongoDB transaction hotspot management, including transaction conflicts, hot document partitioning, and strategic payload distribution.
- **[database-mongodb-transaction-performance](database-mongodb-transaction-performance/SKILL.md)** — Guides the agent on MongoDB transaction performance and the cost of MVCC, transaction limits, transient transaction errors, and driver retry behavior.
- **[database-mongodb-transaction-tuning](database-mongodb-transaction-tuning/SKILL.md)** — Guides the agent on MongoDB transaction tuning, including avoiding unnecessary transactions, ordering transactional operations, and reducing conflict windows.
- **[database-mongodb-transactions](database-mongodb-transactions/SKILL.md)** — Guides the agent on MongoDB ACID transactions, WiredTiger internals, transaction APIs, driver examples, and transaction best practices for Node.js, Python, and Ruby.
- **[database-mongodb-updates](database-mongodb-updates/SKILL.md)** — Guides the agent on MongoDB update operations, including updateOne, updateMany, replaceOne, filters, upsert semantics, and update operator selection.
- **[database-mongodb-upsert-and-merge](database-mongodb-upsert-and-merge/SKILL.md)** — Guides the agent on MongoDB upserts, aggregation-based bulk upserts, cloning data with $merge, and avoiding unnecessary round trips.
- **[database-mongodb-zone-sharding](database-mongodb-zone-sharding/SKILL.md)** — Guides the agent on MongoDB zone sharding, tag ranges, geographic placement, and hot/cold data segregation.
