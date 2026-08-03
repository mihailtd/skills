---
name: database-redis-stack-overview

description: Guides the agent on what Redis Stack is and when to reach for which capability — key-value store, data structure server, multi-model database, or event-driven data platform — before defaulting to a second specialized database.
---

# Redis Stack Overview and Capability Selection

You are an expert in Redis and Redis Stack architecture. When a user is deciding how to model data, whether to add a specialized database (search engine, time-series store, message broker) alongside Redis, or which Redis Stack module solves a given problem, use the following guidance.

- Redis Stack is not a different product from Redis — it is the open source Redis server (BSD-licensed core) plus a set of modules (search/query, JSON, time series, probabilistic data structures, programmability) bundled into one binary, plus RedisInsight (a GUI/data browser) and first-class client SDKs. It ships under a dual license (RSALv2/SSPL) and is free to self-host for development and production, as a Docker image or a native/binary install.
- Think of Redis Stack's scope as four progressively broader layers, and match the user's problem to the narrowest layer that solves it:
  1. **Key-value store** — simple GET/SET by key. This is the original Memcached-alternative use case: sessions, tokens, feature flags.
  2. **Data structure server** — native Hashes, Sets, Sorted Sets, Lists, Bitmaps/Bitfields, Streams, HyperLogLog, geospatial indexes. Reach for this layer whenever the *shape* of the data (an object, a ranked leaderboard, a unique collection, a time-ordered event log) maps naturally onto one of these structures — modeling against the native structure avoids the impedance mismatch of forcing everything through plain strings/JSON blobs.
  3. **Multi-model database** — secondary indexing/search (RediSearch), JSON documents, time series, all coexisting with the core data structures in the same server and keyspace.
  4. **Data platform** — stream processing and event-driven programmability (Functions/triggers) on top of the above, letting Redis react to data changes instead of only serving reads/writes passively.
- Before recommending a second specialized database (Elasticsearch for search, InfluxDB/Prometheus for metrics, a message broker for events), check whether the Redis Stack capability already installed covers the need — consolidating onto one data platform reduces the operational surface (fewer systems to deploy, monitor, and keep consistent) at the cost of being tied to Redis Stack's specific feature set for that workload.
- Don't over-consolidate reflexively either: a dedicated system is still the right call when its specialized feature set (e.g. Elasticsearch's relevance tuning ecosystem, a purpose-built OLAP warehouse) meaningfully exceeds what the Redis Stack module offers for a demanding workload. Evaluate the actual requirement, don't assume "it's all in Redis now" is automatically better.

## Capability-to-Problem Map

Use this to route a requirement to the right skill/module instead of forcing everything through plain key-value strings:

| Need | Redis Stack capability | Skill |
|---|---|---|
| Store/retrieve an object by its ID | Hash | `database-redis-data-modeling-fundamentals` |
| Filter/search on non-key attributes, full-text, faceted, or aggregate (GROUP BY-style) queries | RediSearch | `database-redis-search-indexing` |
| Store nested/hierarchical objects, arrays, or need atomic updates on a nested field | JSON documents | `database-redis-json-documents` |
| Track metrics or events over time and query by time range | Time series | `database-redis-time-series` |
| Approximate answers over huge cardinalities (unique counts, top-k, membership) where exact accuracy isn't required | Probabilistic data structures | `database-redis-probabilistic-data-structures` |
| React to data changes, run scheduled maintenance, or execute logic close to the data | Functions / keyspace triggers | `database-redis-scripting-functions` |
| Need to filter/search but RediSearch isn't available (older Redis, no Stack modules) | Manual secondary indexing | `database-redis-manual-secondary-indexing` |
| Survive a primary instance failure without manual intervention | Replication + Sentinel | `database-redis-high-availability` |
| Scale horizontally across multiple nodes | Cluster (sharding) | `database-redis-cluster-sharding` |
| Restrict what different clients/services can do against the database | ACL | `database-redis-access-control` |
| Encrypt client or internal (replication/cluster) traffic | TLS / mTLS | `database-redis-tls-security` |
| Understand Redis's consistency/durability guarantees as a primary datastore | BASE/ACID mapping | `database-redis-primary-database-guarantees` |

## Getting a Redis Stack Instance

- Fastest local start: `docker run -d --name redis-stack-server -p 6379:6379 redis/redis-stack-server:latest`. See `database-redis-environment-setup` for the full range of install methods (Docker variants, native packages) and client library setup for Python, JavaScript/TypeScript, and Go.
- Verify Stack modules are present (not just core Redis) before relying on `FT.*`, `JSON.*`, `TS.*`, or probabilistic commands — plain `redis-server` (without Stack) does not include them.
