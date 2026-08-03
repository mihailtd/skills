---
name: python-lazy-vs-eager-initialization
description: Instructs the agent on deciding between lazy and eager initialization for expensive startup resources (database connection pools, cache warming, downstream client setup) — eager initialization in a FastAPI lifespan handler pays the cost once at startup and, critically, turns a broken dependency into a failed readiness probe that halts a Kubernetes rolling deployment before it reaches real traffic, while lazy initialization defers the cost onto the first N requests (N being the pool size, not just the first request) and lets a broken dependency go undetected until the app is already serving production traffic.
---

# Python Lazy vs. Eager Initialization Guidelines

You are an expert Python backend developer specializing in service startup
behavior. When asked to set up a database connection pool, warm a cache, or
initialize a downstream client, choose deliberately between lazy and eager
initialization rather than defaulting to whichever is more convenient to
write.

## 1. The Core Tradeoff: Startup Time vs. First-Request Latency

Every expensive resource — a database connection pool, a cache prefill, a
downstream service client — has to be initialized somewhere. There are only
two choices for *when*:

- **Eager**: pay the cost once, at application startup, before the app
  accepts any traffic.
- **Lazy**: defer the cost to the first time the resource is actually used,
  paid by whichever request triggers it.

The cost doesn't disappear either way — it only moves. For a connection pool
sized to N connections, lazy initialization doesn't defer the cost to just
the first request; it's paid across the first N requests, since each of the
first N concurrent callers triggers creation of its own connection before
the pool is warm. Don't reason about this as "first request slow, rest
fine" — it's "first N requests slow."

## 2. Eager Initialization Turns a Broken Dependency into a Failed Deployment, Not a Production Incident

This is the decisive factor for services running behind a rolling deployment
(Kubernetes or equivalent), and it outweighs the startup-time cost in most
cases: **when** a problem is detected matters as much as **whether** it's
detected.

- With **eager** initialization performed before the app signals it's ready
  to serve traffic, a broken dependency (wrong credentials, an
  unreachable database, a downstream service outage, a programming bug in
  the cache-warming query) causes the readiness probe to fail. Kubernetes
  never routes traffic to the broken pod, and a rolling deployment halts
  before replacing the last healthy old pod — the bad version never serves
  a single real request.
- With **lazy** initialization, the same broken dependency has no effect on
  the readiness probe — the app reports ready immediately, because nothing
  has actually tried to use the resource yet. The rollout proceeds, healthy
  old pods get terminated, and the failure only surfaces once real traffic
  hits the new pods and triggers the lazy initialization — by which point
  there may be no healthy old version left to fall back to.
- Treat this as the default tie-breaker: initialize eagerly, inside the
  startup phase, for anything whose failure should block a deployment
  rather than degrade production traffic.

## 3. Implementing Eager Initialization with FastAPI's Lifespan

Perform eager initialization in the `lifespan` async context manager, and
don't `yield` — meaning don't let the app start accepting traffic or pass
its readiness probe — until it completes.

```python
from collections.abc import AsyncGenerator
from contextlib import asynccontextmanager

from fastapi import FastAPI
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    engine = create_async_engine("postgresql+asyncpg://user:pass@host/db")
    app.state.session_factory = async_sessionmaker(engine, expire_on_commit=False)

    # Fail fast: verify connectivity now, not on the first request.
    async with app.state.session_factory() as session:
        await session.execute(select(1))

    await warm_required_cache(app)

    yield  # readiness probe only turns green after this point

    await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

Anything constructed before `yield` runs during startup and gates readiness.
Don't move this construction into a per-request `Depends` function that
lazily creates the engine on first call — that reintroduces exactly the
deferred-failure-detection problem this pattern exists to avoid.

## 4. When Lazy Genuinely Wins: Hybrid, Not All-or-Nothing

Lazy initialization is the right call specifically when startup time is
under real pressure (a hard cold-start SLA, a serverless/scale-to-zero
environment where every millisecond of startup is billed or latency-visible)
and the resource being deferred is both non-critical and cheap enough that a
slow first hit is acceptable — an optional, rarely-used cache branch, for
example.

Decide per-resource, not application-wide:

- **Eager** for anything whose failure should block a deployment: the
  primary database connection, required configuration, credentials that
  must be valid for the app to function at all.
- **Lazy** for anything genuinely optional where a slow or even failed first
  access degrades gracefully rather than taking down the request — an
  optional secondary cache, a non-critical enrichment call.

A service that's "eager for the primary DB pool, lazy for one optional
cache" is a deliberate, defensible design — treat it as such rather than
picking one strategy reflexively for every resource in the application.

## Related guidance

- **architecture-simplicity** (package `architecture`) — the general
  discipline behind point 4's per-resource decision: pick the strategy each
  specific resource actually needs, rather than applying one global default
  (all-eager or all-lazy) to every resource in the application regardless of
  fit.
- **python-fastapi-project-structuring** (package `python-fastapi`) — where
  the `lifespan` handler and its startup dependencies fit in a larger
  FastAPI project layout.
