---
name: database-redis-keyspace-triggers

description: Guides the agent on Redis Stack keyspace triggers (registerKeySpaceTrigger) — JavaScript functions that fire automatically on changes to keys matching a prefix, without any client having to call them — including the critical trigger-timing gotcha inside MULTI/EXEC transactions and Lua scripts, and the onTriggerFired escape hatch that fixes it.
---

# Redis Stack Keyspace Triggers

You are an expert in Redis Stack's keyspace triggers. When a user needs Redis to react automatically to data changes — not poll for them, not rely on every code path remembering to call some logic — guide them to `registerKeySpaceTrigger` instead of a manually-invoked function (see `database-redis-javascript-functions`), and make sure they understand the transaction-timing gotcha before they rely on triggers for anything transaction-heavy.

- **A keyspace trigger is a standing subscription, not a one-time call** — register it once, and it fires automatically every time a matching event happens on a matching key, for as long as the library stays loaded, with no client needing to remember to invoke anything:

  ```javascript
  redis.registerKeySpaceTrigger("del_logger", "user:", function(client, data) {
      if (data.event == 'del') {
          client.call("INCR", "removed");
          redis.log("A user has been removed");
      }
  });
  ```

  This guarantees the side effect happens exactly when the triggering event happens, regardless of which client or code path caused it — the right tool for audit trails, cache invalidation, or derived-data maintenance that must never be silently skipped because some caller forgot to call it explicitly.
- **The trigger fires on the *command*, and `data` tells you what happened — but for a deletion, only the key name is visible, not the deleted value.** By the time a `del` event fires, the data is already gone; if the deleted value itself is needed by the trigger logic, it has to be captured *before* the delete happens elsewhere, not recovered from the trigger.
- **Prefer a keyspace trigger over a manually-invoked function specifically when the requirement is "this must always happen, automatically, no matter which client made the change"** — see `database-redis-javascript-functions` for the on-demand alternative when logic only ever needs to run when explicitly asked.

## The Transaction-Timing Gotcha

- **Outside a transaction, each write fires its own trigger invocation, seeing that write's own value** — straightforward, one event per write:

  ```
  SET captured:123 maria   -- trigger fires, sees "maria"
  SET captured:123 john    -- trigger fires, sees "john"
  ```

- **Inside `MULTI`/`EXEC` (or a Lua script/function), notifications fire only once execution completes — at `EXEC` time, not once per command inside it.** Every notification for keys touched during that atomic block sees the *final* state after the whole block completed, not the value at the moment each individual command ran:

  ```
  MULTI
  SET captured:123 maria
  SET captured:123 john
  EXEC
  -- trigger fires twice, but both invocations read "john" — the "maria" write is invisible to the trigger
  ```

  This is a direct, easy-to-miss consequence of atomic execution: from any outside observer's perspective (including the trigger itself, if it reads the key fresh), the transaction's intermediate states never existed — only the final state is ever visible. **Don't assume a trigger sees every write; it only reliably reflects the value at the moment the trigger callback actually runs**, which for a `MULTI`/`EXEC` block is *after* the whole block finished.
- **Fix this with the optional `onTriggerFired` callback, which runs immediately after each individual key change — inside the atomic block, not after it — and can capture that moment's value into the `data` argument passed to the main trigger function:**

  ```javascript
  redis.registerKeySpaceTrigger("trigger_test", "captured:", function(client, data) {
      redis.log(data.value);
  }, {
      onTriggerFired: (client, data) => {
          data.value = client.call('GET', data.key);
      }
  });
  ```

  With this in place, the same transaction now produces two log entries reflecting the actual sequence — `"maria"` then `"john"` — because `onTriggerFired` captured each value at the moment it was actually set, before the next write in the same transaction overwrote it.
- **Use `onTriggerFired` whenever a trigger's logic needs to see every individual change within a transaction/script, not just the end state** — an audit log that must record every write (not just the last one per key per transaction) is exactly the case this fixes. If only the final state ever matters, the plain trigger (without `onTriggerFired`) is simpler and sufficient.

## Related Behavior

- **Key expiration and eviction can also fire keyspace triggers**, but because both are inherently probabilistic in *when* they actually happen (Redis doesn't guarantee an expired key is removed at the exact millisecond its TTL elapses), don't rely on a trigger for expiration/eviction to fire at a precise time — treat it as "eventually, close to when it happened," not a real-time guarantee.

## Code Example

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

TRIGGER_LIBRARY = """
#!js api_version=1.0 name=triggers
redis.registerKeySpaceTrigger("del_logger", "user:", function(client, data) {
    if (data.event == 'del') {
        client.call("INCR", "removed");
        redis.log("A user has been removed");
    }
});

redis.registerKeySpaceTrigger("captured_logger", "captured:", function(client, data) {
    redis.log(data.value);
}, {
    onTriggerFired: (client, data) => {
        data.value = client.call('GET', data.key);
    }
});
"""

async def load_trigger_library() -> None:
    await client.execute_command("TFUNCTION", "LOAD", "REPLACE", TRIGGER_LIBRARY)
```
