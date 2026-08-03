---
name: database-redis-javascript-functions

description: Guides the agent on Redis Stack's JavaScript Functions (TFUNCTION LOAD/TFCALL/TFCALLASYNC, running on the V8 engine) — library anatomy, calling Redis commands via the client object, synchronous vs asynchronous (coroutine) execution and the block() pattern for atomic sections, and cluster-aware remote functions (registerClusterFunction/runOnShards/runOnKey) — the on-demand-execution half of Redis Stack's JS programmability, distinct from event-driven triggers.
---

# Redis Stack JavaScript Functions

You are an expert in Redis Stack's JavaScript Functions (built on the V8 engine, via the redisgears_2 module). When a user needs server-side logic invoked on demand — not Lua, and not reacting to events (see `database-redis-keyspace-triggers`/`database-redis-stream-triggers` for that) — guide them to `TFUNCTION`/`TFCALL`, and specifically to whether the logic needs synchronous atomicity or can run asynchronously to avoid blocking other clients.

- **Choose JavaScript Functions over Lua Functions (`database-redis-scripting-functions`) specifically when the logic needs asynchronous execution, cluster-wide coordination, or event-driven triggering** — all three of these are JavaScript-only capabilities Lua doesn't have. For a purely synchronous, single-shard, on-demand function, either language works; JavaScript's V8 engine and npm-adjacent tooling (bundlers like Webpack for larger projects) may also be a more familiar environment for teams already writing JS elsewhere.
- **A library's header declares the language, API version, and library name** — the API version matters for forward compatibility, since it lets Redis deprecate/evolve the JS API over versions without silently breaking existing libraries:

  ```javascript
  #!js api_version=1.0 name=greetings
  redis.registerFunction('hello_world', function() { return 'Hello world!'; });
  ```

- **Load from a string or a file, and re-loading an existing library name fails unless you say `REPLACE`** — this is a deliberate guard against accidentally clobbering a deployed library:

  ```bash
  redis-cli -x TFUNCTION LOAD < ./greetings.js         # fails if 'greetings' already exists
  redis-cli -x TFUNCTION LOAD REPLACE < ./greetings.js # explicit, atomic whole-library update
  ```

- **Invoke a function by `<library>.<function>`, with the same keys/args positional convention as Lua's `FCALL`:**

  ```
  TFCALL greetings.hello_world 0
  ```

  `TFUNCTION DELETE <name>` removes a library; `TFUNCTION LIST` inspects what's loaded (including version, useful for confirming a deploy actually took effect).
- **Interact with Redis inside a function via the `client` object passed as the function's first argument** — every subsequent parameter is the function's own keys/args, same pattern as Lua's `KEYS`/`ARGV` but delivered as regular JS parameters instead of global tables:

  ```javascript
  redis.registerFunction('create_user', function(client, id, name) {
      client.call('SET', 'user:' + id, name);
      client.call('INCR', 'users');
      return 'User created';
  });
  ```

## Synchronous vs. Asynchronous Execution

- **`registerFunction` (sync) behaves exactly like a Lua function: atomic, and blocks the main thread for its entire duration.** Every write is visible to other clients only once the function completes, and — critically — *no other client can be served at all* while it runs. This inherits the same shared-server risk described in `database-redis-scripting-functions`: an unbounded loop here freezes Redis for everyone, and will eventually be killed by the server's timeout (`lock-redis-timeout`, in milliseconds — raise only with a specific, understood reason, not as a default fix for "my function times out").
- **`registerAsyncFunction` runs in a coroutine on a background thread, freeing the main thread to keep serving other clients while it executes.** This is the mechanism that makes long-running server-side logic *not* a shared-availability risk — reach for it whenever a function's logic is genuinely long-running (a full-keyspace scan, a multi-step batch job) and doesn't need to be a single atomic unit against the rest of the keyspace:

  ```javascript
  redis.registerAsyncFunction('async_loop', async function() { /* ... */ });
  ```

  Invoke with `TFCALLASYNC` instead of `TFCALL`.
- **An async function's `async_client` argument is deliberately more restricted than the sync `client`** — it cannot call Redis commands directly. To touch the keyspace, wrap the specific operation in `async_client.block(...)`, which re-enters an atomic section (exclusive keyspace access) for just that inner block, then releases it:

  ```javascript
  redis.registerAsyncFunction('async_count', async function(async_client) {
      var count = 0, cursor = '0';
      do {
          async_client.block((client) => {
              var res = client.call('scan', cursor, 'MATCH', 'city:*');
              cursor = res[0];
              count += res[1].length;
          });
      } while (cursor != '0');
      return count;
  });
  ```

  This gives you the best of both models for a batch operation: each individual `block()` call is atomic and consistent (a `SCAN` batch can't be corrupted by an interleaved write), but *between* `block()` calls, other clients get a chance to run — unlike the fully-synchronous version, which locks the whole scan from start to finish. Reach for this pattern specifically for "batch process the whole keyspace/a large subset of it" logic, where full-operation atomicity isn't actually required, only per-batch consistency is.
- **Don't assume "async" means "non-blocking for everyone, all the time"** — a `block()` section is still exclusive while it runs, same as a sync function; async only changes what happens *between* those exclusive sections. A function that's one giant `block()` call is functionally equivalent to a sync function, just invoked differently.

## Cluster-Aware Remote Functions

- **`registerClusterFunction` declares a function meant to run on every shard of a cluster, invoked from an originating shard via `runOnShards`** — the originating shard propagates the call, collects every shard's result, and returns a merged response to the caller. This is how you implement a cluster-wide aggregate (e.g. "count strings across the entire cluster") without the client having to know the cluster topology or issue one call per shard itself:

  ```javascript
  redis.registerClusterFunction("stringcounter", async (async_client) => {
      var count = 0, cursor = '0';
      do {
          async_client.block((client) => {
              var res = client.call('SCAN', cursor, 'TYPE', 'string');
              cursor = res[0];
              count += res[1].length;
          });
      } while (cursor != '0');
      return count;
  });

  redis.registerAsyncFunction("nstrings", async (async_client) => {
      let [results, errors] = await async_client.runOnShards("stringcounter");
      if (errors.length > 0) return errors;
      return results.reduce((sum, n) => sum + BigInt(n), BigInt(0));
  });
  ```

- **A cluster function must be written as a coroutine (async)** — it runs in the background on each remote shard, not inline on the originating shard's main thread.
- **Remote cluster functions are read-only — a write attempt from one fails.** This is a hard constraint, not a convention: cluster functions are for cluster-wide *aggregation/inspection*, not cluster-wide *mutation*. If cross-shard writes are genuinely needed, that has to be coordinated differently (e.g. per-key operations issued to the shard that owns each key), not via `registerClusterFunction`.
- **Use `runOnKey` instead of `runOnShards` when the target is one specific shard (the one owning a given key), not every shard** — e.g. "run this remote logic on whichever shard holds `user:123`," rather than a cluster-wide fan-out.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

GREETINGS_LIB = """
#!js api_version=1.0 name=greetings
redis.registerFunction('hello_world', function() { return 'Hello world!'; });
redis.registerFunction('create_user', function(client, id, name) {
    client.call('SET', 'user:' + id, name);
    client.call('INCR', 'users');
    return 'User created';
});
"""

async def load_library(replace: bool = True) -> None:
    # No dedicated redis-py binding for TFUNCTION; issue the raw command.
    cmd = ["TFUNCTION", "LOAD"] + (["REPLACE"] if replace else []) + [GREETINGS_LIB]
    await client.execute_command(*cmd)

async def call_sync_function() -> str:
    return await client.execute_command("TFCALL", "greetings.hello_world", 0)

async def call_async_function(*args) -> str:
    return await client.execute_command("TFCALLASYNC", "greetings.async_loop", 0, *args)
```
