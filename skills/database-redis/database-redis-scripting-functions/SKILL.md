---
name: database-redis-scripting-functions

description: Guides the agent on Redis's Lua server-side programmability — why Redis's single-threaded command execution makes script duration a shared-server concern, EVAL/EVALSHA/SCRIPT LOAD ad hoc scripting versus persistent Redis Functions (FUNCTION LOAD/FCALL, libraries, code reuse, read-only no-writes flags), and the OOM/stale/cluster execution flags — as the foundation JavaScript Functions (database-redis-javascript-functions) build on.
---

# Redis Server-Side Programmability: Lua Scripts and Functions

You are an expert in Redis's Lua-based server-side programmability. When a user needs logic to run atomically, server-side, close to the data — instead of round-tripping data to the client for multi-step operations — guide them to Redis Functions (the modern, persistent mechanism) over raw `EVAL` scripts, and help them understand the single-threaded execution model that makes script duration everyone's problem, not just the caller's.

## Why Execution Duration Is a Shared-Server Concern

- **Redis's command execution is single-threaded, even though the Redis process itself is not.** Separate internal threads handle things like key expiration and I/O, but one main thread executes commands and touches the keyspace, strictly one at a time. This is a deliberate trade-off, not an accident: it eliminates an entire class of concurrency bugs (deadlocks, partial-write races) and gives every command exclusive, uncontended access to the data it touches — including multi-command atomicity via `WATCH`/`MULTI`/`EXEC` — without any locking protocol client code has to implement.
- **The cost of that trade-off: nothing else runs while a script/function is executing.** A long-running or unbounded-loop script (an unbatched full-keyspace `SCAN` inside a script — see `database-redis-manual-secondary-indexing` for why that's risky even client-side) blocks *every other client* for its entire duration, not just the one that called it. This is the single most important operational constraint on everything in this skill: only run logic here with a bounded, measured execution time — don't assume, verify.
- **This is exactly why Redis Stack's secondary indexing (RediSearch — see `database-redis-search-indexing`) exists as an alternative to scanning inside a script**: an index turns an O(N) keyspace scan into a fast, indexed lookup, removing the temptation to write a scripted scan loop in the first place. Reach for indexing before reaching for a script to work around the lack of one.
- **A single Redis Stack instance is bounded by one CPU core** for command execution, as a direct consequence of the single-threaded model. Scaling past that is a clustering concern (Redis Cluster/Enterprise), not something server-side scripting changes — scripting reduces round trips and adds atomicity, it doesn't add parallelism to the main execution path.

## EVAL Scripts vs. Redis Functions

- **`EVAL` runs an ad hoc Lua script inline, with keys/args passed positionally** — access them in the script via `KEYS[n]`/`ARGV[n]`, and call Redis commands via the `redis.call(...)` singleton:

  ```
  EVAL "local name = redis.call('HGET', KEYS[1], 'Name') return ARGV[1]..name" 1 'city:123' 'The city you requested is '
  ```

- **`EVAL` scripts are not persisted by the server — they're re-sent by the client every time**, unless cached with `SCRIPT LOAD` (which returns a SHA1 hash) and re-invoked cheaply via `EVALSHA <hash> ...`. Either way, **the cache doesn't survive a server restart** — scripts are a client-managed convenience, not a database-managed artifact. `SCRIPT FLUSH` clears the cache; `SCRIPT KILL` terminates a long-running script (the emergency exit for exactly the blocking risk described above).
- **`EVAL_RO`/`EVALSHA_RO` enforce read-only execution** — any write command inside the script fails immediately, rather than trusting the script author to not have included one. Use this whenever a script's whole purpose is a read (even a complex, multi-step one) — it's a real safety guarantee, not just documentation of intent:

  ```
  EVAL_RO "local name = redis.call('HGET', KEYS[1], 'Name') redis.call('INCR', 'cnt') return ARGV[1]..name" 1 'city:123' '...'
  -- ERR Write commands are not allowed from read-only scripts.
  ```

- **Redis Functions (`FUNCTION LOAD`/`FCALL`, Redis 7+) supersede raw `EVAL` scripts for anything beyond a one-off** — they exist specifically to fix `EVAL`'s biggest practical problems: scripts aren't persisted/replicated as part of the database, can't call each other, and become hard to manage once a project accumulates more than a couple of them. Default to Functions; reach for a bare `EVAL` only for a genuine one-off you don't need to keep.
- **A function library is a first-class, persistent database object** — stored and replicated alongside the data itself (RDB/AOF), not something the client has to reload after a restart the way an `EVAL`/`EVALSHA` cache does:

  ```
  FUNCTION LOAD "#!lua name=mylib\n redis.register_function('city_fetch_name', function(keys, args) local name = redis.call('HGET', keys[1], 'Name') return args[1]..name end)"
  FCALL city_fetch_name 1 'city:123' 'The city you requested is '
  ```

  `FUNCTION LOAD REPLACE` updates an existing library atomically (the whole library is replaced as one unit, not function-by-function); `FUNCTION DELETE <name>` removes it; `FUNCTION LIST` inspects what's loaded.
- **Functions in the same library can call each other**, enabling real code reuse — something raw `EVAL` scripts fundamentally can't do (each script is an isolated, self-contained blob):

  ```lua
  #!lua name=mylib
  local function myincr()
      redis.call('INCR', 'cnt')
      return 'OK'
  end
  local function city_fetch_name(keys, args)
      local name = redis.call('HGET', keys[1], 'Name')
      myincr()
      return 'The city is '..name
  end
  redis.register_function('city_fetch_name', city_fetch_name)
  ```

- **Mark a function read-only with the `no-writes` flag at registration, and invoke it via `FCALL_RO`** — the same read-only guarantee `EVAL_RO` provides for scripts, but declared once at the function definition instead of trusted per-call:

  ```lua
  redis.register_function{
      function_name='city_fetch_name',
      callback=city_fetch_name,
      flags={ 'no-writes' }
  }
  ```

  A function *without* `no-writes` will reject execution via `FCALL_RO` outright — the flag isn't optional documentation, it's what the read-only call path actually checks.
- **Both scripts and functions execute atomically — this is a real guarantee, not just a performance characteristic.** "The effects either haven't happened yet or have already happened" from any other client's perspective; there's no way to observe a script/function mid-execution. This is what makes them suitable for multi-step logic that must never leave data in a half-updated state, the same guarantee `MULTI`/`EXEC` gives for plain command batches, but with the flexibility of actual control flow (conditionals, loops) that a transaction alone doesn't offer.

## Execution Flags

Both scripts and functions can be tagged with additional flags controlling how they're allowed to execute — declare deliberately, not by copying flags from an unrelated example:

- **`allow-oom`** — permits execution (including write commands) even when the server is out of memory, when it would normally be denied. Reserve for logic that specifically needs to run *during* an OOM condition (e.g. an eviction-assisting cleanup script) — granting this broadly defeats the OOM protection's purpose.
- **`allow-stale`** — permits a script/function to run against a stale replica when the server would otherwise refuse (`replica-serve-stale-data no`). Only appropriate for logic that genuinely doesn't need current data; commands that *do* access data remain restricted even with this flag set.
- **`no-cluster`** — makes the script/function refuse to run at all in Cluster mode. Use for logic that fundamentally assumes single-shard semantics and would behave incorrectly (not just suboptimally) if silently run against one shard of a sharded deployment.
- **`allow-cross-slot-keys`** — permits accessing keys from multiple hash slots in one execution, overriding the normal single-slot restriction. Generally discouraged, especially in Cluster deployments — accessing multiple slots defeats the data-locality assumption clustering is built on. Reach for this only when there's a specific, understood reason cross-slot access is required, not as a default workaround for a "wrong slot" error.

## Lua Scripts vs. Lua Functions vs. JavaScript Functions

| | Lua Scripts (`EVAL`) | Lua Functions (`FCALL`) | JavaScript Functions (`TFCALL`) |
|---|---|---|---|
| Persistence | No — client reloads on restart | Yes — part of the database | Yes — part of the database |
| Execution model | Sync, blocks main thread | Sync, blocks main thread | Sync or async (background thread) |
| Invocation | Client-controlled only | Client-controlled only | Client-controlled, or automatic via triggers |
| Atomicity | Always atomic | Always atomic | Atomic when sync; not atomic across shards when async/cluster-spanning |
| Cluster | Local shard only | Local shard only | Cross-shard (`runOnShards`/`runOnKey`) |
| Closest RDBMS analogy | Ad hoc complex query | Stored procedure | Stored procedure + trigger |

For the JavaScript side of this table — async execution, triggers, and cluster-aware remote functions — see `database-redis-javascript-functions`, `database-redis-keyspace-triggers`, and `database-redis-stream-triggers`.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

LUA_LIBRARY = """
#!lua name=mylib
local function city_by_cc(keys, args)
    local match, cursor = {}, "0"
    repeat
        local ret = redis.call("SCAN", cursor, "MATCH", "city:*", "COUNT", 100)
        for _, keyname in ipairs(ret[2]) do
            local ccode = redis.call("HMGET", keyname, "Name", "CountryCode")
            if ccode[2] == args[1] then
                match[#match + 1] = ccode[1]
            end
        end
        cursor = ret[1]
    until cursor == "0"
    return match
end
redis.register_function('city_by_cc', city_by_cc)
"""

async def load_and_call_function(country_code: str) -> list[str]:
    await client.function_load(LUA_LIBRARY, replace=True)
    return await client.fcall("city_by_cc", 0, country_code)

async def run_adhoc_script(city_key: str, prefix: str) -> str:
    """A genuine one-off — not worth persisting as a library function."""
    script = "local name = redis.call('HGET', KEYS[1], 'Name') return ARGV[1]..name"
    return await client.eval(script, 1, city_key, prefix)

async def cache_and_run_script(city_key: str, prefix: str) -> str:
    """SCRIPT LOAD + EVALSHA — cheaper repeated calls, but not persisted across restarts."""
    script = "local name = redis.call('HGET', KEYS[1], 'Name') return ARGV[1]..name"
    sha = await client.script_load(script)
    return await client.evalsha(sha, 1, city_key, prefix)
```
