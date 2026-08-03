---
name: database-redis-environment-setup

description: Guides the agent on installing and connecting to Redis Stack — the two Docker image variants (with/without RedisInsight), native package managers as a fallback, client library setup for Python/JavaScript-TypeScript/Go, and verifying the server is ready with a health check.
---

# Redis Stack Environment Setup

You are an expert in setting up Redis Stack development and production-like environments, and in connecting to it from Python, JavaScript/TypeScript, and Go. When a user needs to stand up a Redis Stack instance or wire up a client library, use the following guidance.

## Installing Redis Stack

- **Default to Docker for local development and for containerized/production-like environments** — it needs no OS-specific package manager, no manually resolved system library dependencies, and gives a clean, disposable environment. Prefer it over a native package install or a binary download unless there's a specific reason not to (e.g. the target host can't run containers).
- **Pick the image variant based on whether a GUI is wanted, not by habit:**
  - `redis/redis-stack-server` — Redis Stack Server only, no bundled tooling. This is the right choice for containerized/production-like deployments (Kubernetes, CI, servers) where nothing should be running beyond the database itself:

    ```bash
    docker run -d --name redis-stack-server -p 6379:6379 redis/redis-stack-server:latest
    ```

  - `redis/redis-stack` — Redis Stack Server *plus* RedisInsight (the official web-based data browser/visualizer), exposed on an extra port. This is the right choice for a local development machine where a human wants to inspect data visually:

    ```bash
    docker run -d --name redis-stack -p 6379:6379 -p 8001:8001 redis/redis-stack:latest
    ```

    RedisInsight is then reachable at `http://127.0.0.1:8001`.
  - Don't default to the RedisInsight-bundled image for a server/CI environment — it runs an extra process for no benefit there and widens the container's attack surface for nothing gained.
- **Native package managers are a reasonable fallback when Docker genuinely isn't an option** on the target machine (e.g. installing directly onto a bare-metal or VM host that will run Redis Stack as a system service): APT (Debian/Ubuntu) and YUM (RHEL/CentOS) on Linux, Homebrew (`brew install redis-stack/redis-stack/redis-stack`) on macOS. Each adds Redis's own package repository first, then installs `redis-stack-server` from it — consult the current Redis documentation for the exact repository-setup commands for the target distribution, since GPG key and repository URL details do occasionally change between releases.
- **A raw binary download/build is the least convenient option** — useful mainly when neither Docker nor a native package repository is available for the target OS/architecture. If a "library not found" error appears on startup (a common one on Debian-based systems is `libgomp.so.1: cannot open shared object file`), it's a missing system dependency (GCC's OpenMP runtime for that example) — install it via the OS's package manager (`apt-get install libgomp1` / `yum install libgomp`), don't treat it as a Redis Stack bug.

## Client Libraries

Redis Stack speaks the same protocol (RESP) as plain Redis, so any Redis client library works — no Stack-specific client is required, though modern client libraries add convenience wrappers for the Stack modules (`.ft()`, `.json()`, `.ts()`, `.bf()`, as used throughout this package's other skills).

- **Python** — `redis-py`:

  ```bash
  pip install redis
  ```

  Used throughout this package's code examples via `from redis import asyncio as aioredis`.
- **JavaScript/TypeScript** — `node-redis`:

  ```bash
  npm install redis
  ```

  Ships its own TypeScript types — no separate `@types/` package needed for current versions. Prefer this over lower-level RESP clients unless there's a specific reason to bypass the convenience command wrappers (Stack module support, connection pooling, reconnection handling).
- **Go** — `go-redis`, added after the module is initialized:

  ```bash
  go mod init github.com/my/repo
  go get github.com/redis/go-redis/v9
  ```

  Always pin to the `/v9` (or current major version) import path — the module's API has changed across major versions, and mixing versions in a dependency graph causes type-incompatibility errors that are confusing to diagnose.

## Verifying the Server Is Ready

- **`PING` is the canonical readiness check** — reliable, cheap, and doesn't depend on any Stack module being loaded:

  ```bash
  redis-cli -h 127.0.0.1 -p 6379 PING
  ```

  A healthy server replies `PONG`. A server that's still loading its dataset from disk on startup replies with a `LOADING` error instead — treat that as "not ready yet, retry," not as a failure, when scripting a startup health check (e.g. a container orchestrator readiness probe, or a test suite's setup step waiting for the server to come up).
- **Use the client library's ping, not a shell-out to `redis-cli`, when the check needs to run from application code** (e.g. an app-level health endpoint, or a test fixture's setup) — every client library exposes a direct equivalent, avoiding a subprocess dependency.
- **A successful `PING` confirms core Redis is up — it does not confirm Stack modules are loaded.** If the workload depends on `FT.*`/`JSON.*`/`TS.*`/probabilistic commands, verify those specifically (e.g. `FT._LIST` returning without error) rather than assuming a healthy `PING` means the full Stack feature set is available — see `database-redis-stack-overview` for why plain `redis-server` (no Stack modules) is a real possibility to rule out.

## Connecting with Authentication

- **Any Redis instance with a password (or ACL user) configured needs credentials passed at connection time, not just a host and port.** See `database-redis-access-control` for actually configuring ACL users/permissions server-side, and `database-redis-tls-security` for encrypting the connection itself and mutual-TLS certificate-based authentication as an alternative to passwords. Every client library accepts a username/password (or a full connection URL with credentials embedded) — don't hardcode credentials into source; pull them from environment variables or a secrets manager, the same as any other credential.
- **A bare `host:port` connection against a password-protected instance fails on the first command, not at connect time**, in some client libraries — if a "not ready"/auth error shows up only once a command is issued rather than immediately, check whether credentials were supplied at all before debugging further.

## Code Examples

```python
import os
from redis import asyncio as aioredis

# Prefer host/port/username/password kwargs (or REDIS_URL) over hardcoding credentials.
client = aioredis.Redis(
    host=os.environ.get("REDIS_HOST", "127.0.0.1"),
    port=int(os.environ.get("REDIS_PORT", 6379)),
    username=os.environ.get("REDIS_USERNAME"),
    password=os.environ.get("REDIS_PASSWORD"),
)

async def is_ready() -> bool:
    try:
        return await client.ping()
    except Exception:
        return False
```

```typescript
import { createClient } from "redis";

// node-redis accepts credentials either via the url or discrete socket/username/password fields.
const client = createClient({
  username: process.env.REDIS_USERNAME,
  password: process.env.REDIS_PASSWORD,
  socket: {
    host: process.env.REDIS_HOST ?? "127.0.0.1",
    port: Number(process.env.REDIS_PORT ?? 6379),
  },
});
await client.connect();

async function isReady(): Promise<boolean> {
  try {
    const reply = await client.ping();
    return reply === "PONG";
  } catch {
    return false;
  }
}
```

```go
package main

import (
	"context"
	"os"

	"github.com/redis/go-redis/v9"
)

func newClient() *redis.Client {
	return redis.NewClient(&redis.Options{
		Addr:     os.Getenv("REDIS_ADDR"), // e.g. "127.0.0.1:6379"
		Username: os.Getenv("REDIS_USERNAME"),
		Password: os.Getenv("REDIS_PASSWORD"),
	})
}

func isReady(ctx context.Context, rdb *redis.Client) bool {
	_, err := rdb.Ping(ctx).Result()
	return err == nil
}

func main() {
	rdb := newClient()
	_ = rdb // use rdb with isReady(context.Background(), rdb)
}
```
