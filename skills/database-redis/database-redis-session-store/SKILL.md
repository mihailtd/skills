---
name: database-redis-session-store

description: Guides the agent on using Redis as a centralized session store for load-balanced application servers — why sticky sessions and in-process session state break down at scale, which framework integration to reach for instead of hand-rolling one, choosing Hash vs JSON for session content, modeling rich searchable sessions with JSON+RediSearch (cart/geo/collection queries), multi-object sessions, and atomic cross-key TTL refresh.
---

# Redis as a Session Store

You are an expert in web session management architecture. When a user is designing how session state should be stored and shared across application servers, guide them toward a centralized Redis session store instead of in-process session state or sticky-session load balancing, once the architecture has more than one application server.

- **The problem Redis solves here is specific: HTTP is stateless, but the application needs to recognize a returning user across multiple requests, and those requests may land on different application server instances.** Storing session data in the memory of the application server that first created it works only as long as every subsequent request from that user is routed back to the *same* server.
- **Sticky sessions (routing a client to the same server based on a cookie or IP) are a workaround, not a fix, and they don't resolve every problem:**
  - Losing that specific application server (crash, deploy, scale-down) loses every session it was holding, with no recovery path.
  - Session data isn't available to any other service that might need it (analytics, fraud checks, a different microservice needing the user's cart) — it's trapped in one server's memory.
  - Load balancing is less effective when traffic can't be freely redistributed across servers — sticky routing works against the elasticity that horizontal scaling is supposed to provide.
- **Centralizing session state in Redis removes the server-affinity requirement entirely**: any application server can read/write any user's session, because the session lives outside all of them. This is what actually enables free load balancing across a fleet of stateless application servers.
- **Session reads and writes both need to be fast** — reads because the user is waiting on them before their page/response can be returned, and writes because an active session's TTL typically needs extending on every request (a sliding expiration) to keep the user logged in while they're active. This latency requirement is exactly what makes an in-memory store like Redis the right fit, versus a disk-backed database for session state.
- **Prefer an existing framework integration over hand-rolling session serialization/storage logic**, unless there's a concrete reason the existing ones don't fit:
  - Spring: `Spring Session Data Redis`
  - Express (Node.js): `connect-redis`
  - Flask (Python): `Flask-Session` with a Redis backend
  - These handle cookie/ID generation, serialization, and TTL renewal correctly and consistently — reinventing this is easy to get subtly wrong (e.g. forgetting to renew TTL on activity, or serialization bugs that only surface for certain data shapes).
- **A centralized session store unlocks more than "remember the user" once it's searchable** — with RediSearch over session data (see `database-redis-search-indexing`), a business can query live sessions directly: how many active carts contain a given product, the total value of open shopping carts, how many users are mid-checkout right now. This is not something achievable when session state is scattered across individual application server processes.

## Choosing a Data Structure for Session Content

Session data isn't just one flat bag of strings — it typically mixes metadata, nested objects (a cart line item), collections (visited pages), and sometimes geolocation. Pick the storage structure based on what the session actually needs to hold and query, not by default:

- **String** (a serialized blob) — compact, but expensive to serialize/deserialize on every read and write, and **not indexable at all**. Only reasonable when the session is opaque to the database (nothing about it is ever queried server-side) and the application already has fast, reliable serialization in place.
- **Hash** — the conventional, framework-supported default (see the framework integrations above). Indexable and gives direct field-level access, but flat — it can't represent a nested object (a cart line item, a geo coordinate) without serializing that piece into one field's string value, which then can't be queried into.
- **JSON** — nested-structure-native, and just as indexable as Hash. This is the right choice once a session needs to hold genuinely nested data (a shopping cart with per-item cost/quantity, a location, an array of visited pages) *and* some of that nested data needs to be searchable — see the next section.
- **Other structures (Lists, Sets, Sorted Sets, Bitmaps, HyperLogLog, Streams)** — useful for specific session-adjacent needs (a visit history as a List, a unique-page-count via HyperLogLog, a recent-activity Stream) but **not indexable by RediSearch directly**. Use these alongside a Hash or JSON "core" session record, not as a replacement for it, when a session needs one of these specific structures' properties (see "Multi-Object Sessions" below).
- Recurring operations like `HGETALL` on a wide Hash, or `HSCAN`/full-Set-scan just to check membership, are a sign the session's search/access needs have outgrown what direct data-structure commands can do efficiently — that's the point to reach for indexing instead of optimizing the access pattern further.

## Modeling Rich, Searchable Sessions with JSON

When sessions need cross-session search — "which sessions have this item in their cart," "which sessions are near this location," "the most recently active session" — model the session as JSON and index it, rather than trying to flatten everything into Hash fields or querying session-by-session in application code.

- **Mix field types in one schema to match each piece of session data's actual shape**:

  ```
  FT.CREATE session_idx ON JSON PREFIX 1 session:
      SCHEMA $.lastAccessedTime AS lastAccessedTime NUMERIC SORTABLE
             $.creationTime AS creationTime NUMERIC SORTABLE
             $.visited AS visited TEXT
             $.cart[*].itemId AS itemid TAG
             $.location AS loc GEO
  ```

  This single index makes every one of these searchable at once: `NUMERIC SORTABLE` for time-based queries ("most recently active"), `TEXT` for free-text search across visited-page history, `TAG` on an array path (see `database-redis-json-documents`'s multi-value indexing) for exact-match search into nested cart items, and `GEO` for proximity search — without denormalizing the session into separate tables/keys per concern.
- **Cross-session search examples this schema enables directly**:

  ```
  -- every session with a MacBook in the cart
  FT.SEARCH session_idx '@itemid:{MacBook}' RETURN 0

  -- sessions within 40km of a point
  FT.SEARCH session_idx '@loc:[34.5 31.5 40 km]' RETURN 0

  -- most recently created session
  FT.SEARCH session_idx "@creationTime:[-inf +inf]" RETURN 1 creationTime LIMIT 0 1 SORTBY creationTime DESC
  ```

- **Drill into one session's matched detail with a JSONPath filter on the document itself**, once the index has told you *which* session matched — the index finds the session, JSONPath extracts the specific nested value from it:

  ```
  JSON.GET session:a30d0c64-4cad-4088-a9ef-f1889d182df4 '$.cart[?(@.itemId=="MacBook")]'
  ```

- **Fetch only the session fields actually needed with `JSON.GET` and a path, instead of always pulling the whole session document** — this is the same bandwidth discipline as `HMGET` over `HGETALL`, and matters more for sessions than most data because session reads happen on nearly every request:

  ```
  JSON.GET session:28og4f8-2643gf862g4 $.visited
  JSON.GET session:28og4f8-2643gf862g4 $.lastAccessedTime $.visited
  ```

## Multi-Object Sessions

- **A session can span more than one key when part of its data genuinely needs a structure JSON/Hash don't provide** — e.g. the core session record as JSON, plus a page-visit history as a List and a unique-visitor count as a HyperLogLog, all sharing the same session ID as a key prefix:

  ```
  session:3354623a-78fb-45aa-80fb-fc8c7f6afeb5           -- core JSON record
  session:3354623a-78fb-45aa-80fb-fc8c7f6afeb5:history   -- List
  session:3354623a-78fb-45aa-80fb-fc8c7f6afeb5:trackvisits -- HyperLogLog
  ```

  Keep the shared prefix so every key belonging to one session is discoverable together — this is the same `type:id` namespacing convention as `database-redis-data-modeling-fundamentals`, applied to a session that outgrew a single structure.
- **Every key belonging to a multi-object session needs its TTL refreshed together, not just the core record** — refreshing only the JSON record's expiration while an auxiliary List/HyperLogLog key silently outlives (or expires before) it leaves the session in an inconsistent state. Batch the `EXPIRE` calls for every one of a session's keys into a single pipeline (or a transaction if the refresh must be atomic) rather than issuing them as separate round trips — see `database-redis-manual-secondary-indexing`'s pipelining guidance for the general technique, applied here to keep session TTLs in lockstep.

## Defaults

- **Standard sessions** (metadata, simple key-value attributes, maybe one serialized object) — Hash, via a framework integration (Spring Session Data Redis, connect-redis, Flask-Session). This covers the large majority of session use cases with the least custom code.
- **Rich sessions** that need in-session or cross-session search over nested objects, collections, or geolocation — JSON, indexed with RediSearch. Reach for this deliberately once a concrete search requirement exists, not as a default for every session — the added modeling/indexing effort should be paying for a real query need.

## Code Examples

```python
from redis import asyncio as aioredis
import uuid

client = aioredis.from_url("redis://localhost")

async def create_session(user_data: dict, ttl_seconds: int = 1800) -> str:
    """Server generates the session ID; the client only ever sees the opaque token."""
    session_id = str(uuid.uuid4())
    await client.hset(f"session:{session_id}", mapping=user_data)
    await client.expire(f"session:{session_id}", ttl_seconds)
    return session_id

async def get_session(session_id: str) -> dict:
    return await client.hgetall(f"session:{session_id}")

async def touch_session(session_id: str, ttl_seconds: int = 1800) -> None:
    """Sliding expiration: extend TTL on every active request so the session
    doesn't expire out from under an actively browsing user."""
    await client.expire(f"session:{session_id}", ttl_seconds)

async def destroy_session(session_id: str) -> None:
    await client.delete(f"session:{session_id}")

async def create_rich_session(session_data: dict, ttl_seconds: int = 1800) -> str:
    """JSON session: nested cart, visited pages, and location, all in one searchable document."""
    session_id = str(uuid.uuid4())
    await client.json().set(f"session:{session_id}", "$", session_data)
    await client.expire(f"session:{session_id}", ttl_seconds)
    return session_id

async def sessions_with_item_in_cart(item_id: str) -> list[str]:
    """Cross-session search enabled by TAG-indexing $.cart[*].itemId."""
    result = await client.execute_command(
        "FT.SEARCH", "session_idx", f"@itemid:{{{item_id}}}", "RETURN", 0,
    )
    return result

async def lazy_load_session_fields(session_id: str, *paths: str) -> dict:
    """Fetch only the fields needed for this request, not the whole session document."""
    return await client.json().get(f"session:{session_id}", *paths)

async def touch_multi_object_session(session_id: str, ttl_seconds: int = 1800) -> None:
    """Refresh TTL across every key belonging to a multi-structure session, atomically."""
    async with client.pipeline(transaction=True) as pipe:
        pipe.expire(f"session:{session_id}", ttl_seconds)
        pipe.expire(f"session:{session_id}:history", ttl_seconds)
        pipe.expire(f"session:{session_id}:trackvisits", ttl_seconds)
        await pipe.execute()
```
