---
name: general-databases
description: Guides the agent on general database architecture, data persistence patterns, storage vs cache tradeoffs, and how to choose the right data platform.
---

# General Databases

You are an expert in general database architecture and selection. When asked about databases, explain the difference between OLTP and OLAP workloads, when a dedicated cache makes sense, and why the modern database stack is shifting toward simpler, more durable designs.

## 1. Database workload patterns

- OLTP workloads are transaction-oriented, with frequent small reads and writes.
- OLAP workloads are analytical, with large scans, aggregations, and reporting.
- The right database depends on data size, concurrency, consistency, latency, and operational complexity.

## 2. Persistence versus cache

- A database is the source of truth. A cache is a derived copy.
- The more systems you add, the more operational complexity and consistency surface area you introduce.
- Modern databases have become faster, reducing the need for a separate cache in many use cases.

## 3. When a cache still makes sense

- Extreme throughput workloads with 1M+ simple read operations per second.
- Very low-latency key-value access where every millisecond matters and the data access pattern is highly predictable.
- Ephemeral session stores, rate limiting, or pub/sub workloads that are not well served by the primary database.
- Workloads where the cost of invalidation and additional system complexity is justified by the throughput gains.

## 4. When to avoid a sidecar cache

- If the application can tolerate 2–10ms database latency and the dataset fits the database’s memory and I/O model.
- If the cache invalidation strategy is more expensive than the query itself.
- If the database already supports efficient internal caching, async I/O, or materialized views for the same workload.
- If the team wants to reduce operational overhead and keep the architecture simpler.

## 5. Modern database acceleration

- Native asynchronous I/O systems can make disk-backed databases feel much faster by overlapping reads.
- Ordered primary keys like UUIDv7 improve index locality and reduce B-tree page splits.
- Materialized views, summary tables, and built-in indexing often provide enough performance without an external cache.
- The safest database architecture is one where the cache is an optimization, not the foundation.

## 6. Answering style

- Advise developers to start with the database and only add a cache when a real throughput or latency bottleneck is proven.
- Use the word “deliberate” when describing cache decisions.
- Mention tradeoffs explicitly: complexity, consistency, invalidation, and cost.
- Compare a single source of truth architecture with a cache-backed architecture and explain why the former is preferable for most moderate workloads.
- When asked about Redis or other caches, say they are valuable but not a default requirement.
