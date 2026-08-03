---
name: database-redis-access-control

description: Guides the agent on Redis's Access Control List (ACL) system — requirepass/AUTH as the baseline, ACL SETUSER for per-user command/key restrictions, command categories (ACL CAT) for coarse-grained rules, and the full rule syntax (command/category allow-deny, key-prefix read/write scoping) for fine-grained least-privilege access.
---

# Redis Access Control (ACL)

You are an expert in Redis's ACL system. When a user needs to restrict what different clients/services/people can do against a Redis instance — beyond a single shared password — guide them to ACL users and rules instead of relying on network-level trust alone (see `database-redis-tls-security` for the network/transport layer this complements, not replaces).

- **The baseline, single-password mode is `requirepass`/`AUTH`** — every client authenticates as the implicit `default` user with full permissions:

  ```
  requirepass <yourSecretPassword>
  ```

  ```
  AUTH <yourSecretPassword>
  ```

  This is better than nothing, but it's all-or-nothing: any client with the password can run any command against any key. Real least-privilege access needs actual ACL users.
- **Create a user with `ACL SETUSER`, and don't forget the `on` directive** — a user created without it exists but is disabled and can't authenticate at all:

  ```
  ACL SETUSER mirko >ortensi on
  ```

  (`>ortensi` sets the password; the user then authenticates via `AUTH mirko ortensi`.)
- **Start from command categories for coarse-grained rules before reaching for individual command allow/deny lists** — categories cover the common "this service should only read" / "this service should only write" shape with one rule instead of enumerating every command:

  ```
  ACL CAT
  -- keyspace, read, write, set, sortedset, list, hash, string, bitmap,
  --   hyperloglog, geo, stream, pubsub, admin, fast, slow, blocking,
  --   dangerous, connection, transaction, scripting
  ```

  ```
  ACL SETUSER luigi >fugaro on +@read ~*
  ACL SETUSER mirko on +@write ~*
  ```

  Here `luigi` can run any read-only command against any key, and `mirko` can run any write command against any key — each rejected outright (`NOPERM`) for anything outside their category, including commands that might seem harmless (a write-only user genuinely cannot run `KEYS`/`GET`, since those are read commands).
- **Inspect what's actually configured with `ACL LIST` (all users, compact form) or `ACL GETUSER <name>` (one user, structured detail)** — don't assume a rule did what you intended; verify it, especially before relying on it for anything security-sensitive:

  ```
  ACL LIST
  1) "user default on nopass sanitize-payload ~* &* +@all"
  2) "user luigi on sanitize-payload #hashed_password ~* +@read"
  3) "user mirko on sanitize-payload #hashed_password ~* +@write"
  ```

  **Check the `default` user's configuration specifically** — by default it has no password and full permissions (`nopass ~* &* +@all`), which is a real exposure if anything can reach Redis without going through your intended ACL users. Either lock down `default` (give it a password, restrict its permissions) or disable it outright once real users are in place, rather than leaving it as an unintended full-access backdoor.

## Rule Syntax Reference

| Rule | Effect |
|---|---|
| `+command` / `-command` | Allow / disallow one specific command |
| `+@category` / `-@category` | Allow / disallow every command in a category (see `ACL CAT`) |
| `allcommands` | Alias for `+@all` |
| `nocommands` | Alias for `-@all` |
| `~keyPrefix` | Allow access (per whatever command rules apply) to keys matching this prefix/pattern |
| `%R~keyPrefix` | Read-only access to keys matching this prefix |
| `%W~keyPrefix` | Write-only access to keys matching this prefix |
| `%RW~keyPrefix` | Read-write access to keys matching this prefix (equivalent to plain `~keyPrefix` combined with both read and write command rules) |
| `allkeys` | Alias for `~*` |

- **Combine a command rule with a key rule to express "this user may do X, but only to keys matching Y"** — the two dimensions (which commands, which keys) are independent and both need to be set; a user with `+@read` but no key rule can't actually read anything, since no keys are in scope.
- **Prefer the narrowest rule that satisfies the actual requirement** — `~user:*` instead of `~*` when a service only ever needs to touch `user:`-prefixed keys, `%R~cache:*` instead of `~cache:*` when it only ever reads them. This is the practical form of least-privilege: a compromised or buggy client with an overly broad ACL user can do far more damage than one scoped tightly to what it actually needs.
- **Rules not covered here** (enabling/disabling a user with `on`/`off`, setting/removing a password with `>`/`<`) exist too — consult the current Redis ACL documentation for the complete rule set and any version-specific additions rather than assuming this reference is exhaustive.

## Code Examples

```python
from redis import asyncio as aioredis

# Connect as a specific ACL user, not the default user.
client = aioredis.Redis(host="localhost", port=6379, username="mirko", password="ortensi")

async def create_read_only_user(admin_client, username: str, password: str, key_prefix: str) -> None:
    await admin_client.execute_command(
        "ACL", "SETUSER", username, f">{password}", "on", "+@read", f"~{key_prefix}*",
    )

async def create_write_only_user(admin_client, username: str, password: str, key_prefix: str) -> None:
    await admin_client.execute_command(
        "ACL", "SETUSER", username, f">{password}", "on", "+@write", f"~{key_prefix}*",
    )

async def inspect_user(admin_client, username: str) -> dict:
    """Verify what a user can actually do before trusting the rule did what was intended."""
    return await admin_client.acl_getuser(username)
```
