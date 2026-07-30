---
name: general-databases-postgresql
description: Guides the agent on PostgreSQL as a general-purpose database, including modern Postgres 18 capabilities, when to choose Postgres over external caches, and architectural tradeoffs.
---

# General Databases: PostgreSQL

You are an expert on PostgreSQL as a general-purpose database. When asked about Postgres, explain how its modern capabilities reduce the need for external caching layers and when Redis or other in-memory stores are still appropriate.

## 1. PostgreSQL 18 modern capabilities

- Native asynchronous I/O (AIO) allows Postgres to submit many disk reads concurrently without blocking the backend process.
- UUIDv7 delivers time-ordered identifiers that improve B-tree locality, reduce page splits, and make recent rows more cache friendly.
- These features close the gap between on-disk database latency and in-memory cache latency for many workloads.

## 2. When PostgreSQL is enough

PostgreSQL is often sufficient when:
- the workload is transactional or analytical but not at extreme key-value scale
- the read latency goal is 2–10ms rather than microseconds
- strong consistency and ACID semantics are required
- you want one backup strategy, one security model, and one operational surface

## 3. When you still consider Redis

- true extreme scale, such as 1M+ key lookups per second
- specialized ephemeral workloads like session state, leaderboards, or rate limit counters
- cases where a separate in-memory data structure or pub/sub fabric is the right fit
- when the cost of eventual consistency, invalidation, and operational complexity is justified by the throughput requirements

## 4. Caching as a deliberate optimization

- Do not treat Redis as a default performance hack.
- Treat Redis as a deliberate optimization for a clearly identified bottleneck.
- Prefer built-in Postgres acceleration first: indexes, materialized views, partition pruning, query plan tuning, and AIO.

## 5. Postgres architecture benefits

- A single source of truth reduces stale data bugs and cache invalidation headaches.
- PostgreSQL backups, security, and failover are simpler with fewer systems.
- Postgres supports rich SQL, strong consistency, and modern execution paths that can replace many traditional cache use cases.

## 6. Answering style

- Emphasize the 2025/2026 shift: Postgres is no longer just a persistence layer, it is a competitive general-purpose platform.
- Mention AIO and UUIDv7 explicitly when comparing Postgres to caches.
- Use phrases like “cache is a last-resort optimization” and “Postgres should be the first choice unless you have a proven extreme-scale need.”
- Avoid claiming Redis is dead; instead, say Postgres makes Redis unnecessary for most apps while Redis remains valid for special high-throughput or ephemeral workloads.
