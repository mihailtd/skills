---
name: database-redis-transactions

description: Guides the agent on Redis transactions — MULTI/EXEC/DISCARD, WATCH-based optimistic locking as the actual isolation mechanism (Redis has no lock manager), what does and doesn't get rolled back on error, and atomic multi-key commands (MSET, SADD, HSET, JSON.SET) as a lighter-weight alternative to a full transaction.
---

# Redis Transactions

You are an expert in Redis's transaction model. When a user needs multiple commands to execute as one atomic unit, or needs to protect a read-then-write sequence from a concurrent modification, guide them to the correct mechanism — plain atomic commands, `MULTI`/`EXEC`, or `WATCH`-based optimistic locking — and make sure they understand precisely what Redis does and doesn't guarantee, since it differs from a traditional RDBMS transaction in specific, important ways.

- **`MULTI` queues commands; `EXEC` runs them all atomically; `DISCARD` cancels the queue without running anything:**

  ```
  MULTI
  SET greetings hello
  DISCARD    -- nothing executed; GET greetings still returns nil
  ```

  Between `MULTI` and `EXEC`, commands are queued, not executed — this is why a `WATCH` (below) can still be meaningfully checked at `EXEC` time.
- **A syntax error at queue time aborts the entire transaction automatically — nothing in it runs, not even the commands that were valid.** Redis checks each command's syntax as it's queued, and an unknown command or malformed syntax anywhere in the queue poisons the whole transaction:

  ```
  MULTI
  SSET greetings hello        -- ERR unknown command 'SSET' ...
  SET greetings hello         -- QUEUED (still accepted)
  EXEC                        -- (error) EXECABORT Transaction discarded because of previous errors.
  ```

- **A *runtime* error (a command that's syntactically valid but semantically wrong, like using the wrong data-type command against a key) is a completely different story: it does NOT abort the rest of the transaction.** Every other command in the batch still runs — this is the single most important, most surprising thing to understand about Redis transactions if you're used to an RDBMS's automatic rollback-on-error:

  ```
  MULTI
  SET hola mundo         -- QUEUED
  SADD greetings ciao    -- QUEUED (greetings is actually a List, not a Set)
  EXEC
  1) OK
  2) (error) WRONGTYPE Operation against a key holding the wrong kind of value
  ```

  After this, `hola` is genuinely set to `"mundo"` — the transaction is **partially applied**, by design. Redis doesn't pre-validate command semantics against actual key types before executing a transaction (that check would cost performance on every transaction to catch what should be a bug caught in development/testing, not a runtime condition to defend against). **Application code must check each element of the `EXEC` result array and handle partial application explicitly** — never assume a transaction either "fully happened" or "fully didn't."
- **`WATCH` is the actual isolation mechanism — without it, `MULTI`/`EXEC` provides atomicity but not isolation from concurrent changes.** Redis has no lock manager; a `MULTI`/`EXEC` block doesn't block other clients from reading or writing the same keys while it's queued. A read-then-write pattern is vulnerable to a classic race without `WATCH`:

  ```
  -- session A                          -- session B (interleaved)
  SET greeting hello
  MULTI
  APPEND greeting " world"
                                          SET greeting ciao
  EXEC                                  -- (integer) 10 — appended onto "ciao", not "hello"!
  GET greeting                          -- "ciao world", not "hello world"
  ```

  `WATCH` turns this into optimistic locking: watch the key(s) the transaction's logic depends on, and if any watched key changes between the `WATCH` and the `EXEC`, the whole transaction is aborted (returns `nil`) instead of running against stale data:

  ```
  WATCH greeting
  MULTI
  APPEND greeting " world"
  EXEC    -- (nil) if `greeting` changed since WATCH — the append never happened
  ```

  **Always `WATCH` every key a transaction's logic reads before deciding what to write**, whenever that decision depends on the key's current value — an unwatched read-modify-write inside `MULTI`/`EXEC` is not actually protected from the race it looks like it's protected from.
- **A transaction is confined to keys reachable from a single shard.** In a clustered deployment (Redis Cluster/Enterprise), there is no cross-shard/cross-slot distributed transaction — this is a deliberate design boundary for performance, not a current limitation expected to be lifted. Keep transactional logic scoped to keys that are guaranteed to land on the same shard/slot — see `database-redis-cluster-sharding` for hash tags, the technique for deliberately forcing related keys into the same slot.
- **For a single change to a single collection, an atomic command is simpler and just as safe as wrapping it in `MULTI`/`EXEC`** — many commands are already atomic on their own, with no transaction wrapper needed:

  ```
  HSET document:123 title "Talking about ACIDity" content "Variadic commands are atomic"
  JSON.SET document:123 $ '{"title": "...", "content": "..."}'
  MSET user:123 "John Smith" user:123:address "..."
  ```

  Multi-key commands like `MSET`, `SADD`/`SINTERSTORE`/`SUNIONSTORE`/`SDIFFSTORE`, `ZUNIONSTORE`/`ZINTERSTORE`/`ZDIFFSTORE`, `SMOVE`, `RPOPLPUSH`/`BRPOPLPUSH`, `SORT`, and `BITOP` are all atomic without any `MULTI`/`EXEC` wrapper — reach for `MULTI`/`EXEC` specifically when the atomic unit spans *multiple separate commands*, not when one already-atomic command does the whole job.
- **Lua scripts, Redis Functions, and JavaScript Functions all execute atomically too** (see `database-redis-scripting-functions`/`database-redis-javascript-functions`) — the same transactional-behavior discussion applies to them: no other client can observe a partial execution, but a runtime error partway through a script doesn't automatically roll back what already ran, same as `EXEC`.
- **A crash timed exactly at the edge of a transaction can still produce a corrupted-on-disk transaction, even with the safest persistence settings** — this is a persistence-layer concern, not a transaction-logic one; see `database-redis-persistence-durability` for how Redis writes transactions to the AOF and how to recover from a truncated one.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def atomic_batch(user_id: str, address: str) -> None:
    """Multiple independent writes as one atomic unit — MULTI/EXEC."""
    async with client.pipeline(transaction=True) as pipe:
        pipe.set(f"user:{user_id}", "John Smith")
        pipe.set(f"user:{user_id}:address", address)
        await pipe.execute()

async def optimistic_append(key: str, suffix: str, max_retries: int = 3) -> bool:
    """WATCH-based optimistic locking: retry if the key changed underneath us."""
    for _ in range(max_retries):
        async with client.pipeline(transaction=True) as pipe:
            await pipe.watch(key)
            current = await pipe.get(key)
            pipe.multi()
            pipe.set(key, (current or b"").decode() + suffix)
            try:
                await pipe.execute()
                return True
            except aioredis.WatchError:
                continue  # key changed between WATCH and EXEC — retry
    return False

async def check_partial_transaction_result(user_id: str, address: str) -> list:
    """Runtime errors don't abort the batch — every result must be checked individually."""
    async with client.pipeline(transaction=True) as pipe:
        pipe.set(f"user:{user_id}", "John Smith")
        pipe.sadd(f"user:{user_id}", "wrong-type-example")  # may fail if the key is a String
        results = await pipe.execute(raise_on_error=False)
    return results  # inspect each element — success and failure can coexist
```
