---
name: database-redis-tls-security

description: Guides the agent on network-level Redis security — never binding Redis to a public interface, TLS encryption for client and internal (replication/cluster) traffic, the certificate files required, and mutual TLS (mTLS) as a certificate-based alternative to password authentication.
---

# Redis Network Security (Bind, TLS, mTLS)

You are an expert in Redis's network-level security. When a user is deploying Redis anywhere beyond a trusted local machine, guide them through the two layers that actually matter — never exposing Redis directly to untrusted networks, and encrypting the traffic that is allowed to reach it — before any authentication/ACL configuration (see `database-redis-access-control`) is even relevant.

- **The first and most important control is network exposure, not encryption: Redis should never be directly reachable from an untrusted network, full stop.** Clients (the application's *backend*, never a frontend directly) should reach Redis through a controlled network path — an internal network, a VPC, a backend service acting as a mediator — not have Redis's port exposed publicly. This is a more fundamental protection than TLS: an encrypted connection to a Redis instance sitting on the open internet is still a Redis instance sitting on the open internet.
- **`bind` controls which network interface Redis listens on — keep it internal:**

  ```
  bind 127.0.0.1
  ```

  The default (loopback only) is deliberately restrictive. If Redis needs to be reachable from other hosts (a real deployment, not a single-machine dev setup), bind to a specific internal/private interface address — never `0.0.0.0` or a public IP, which would defeat the entire point of this control.

## TLS: Encrypting Traffic

- **TLS protects two distinct kinds of Redis traffic, and each needs its own configuration flag turned on** — enabling TLS for client connections doesn't automatically also encrypt replication or cluster gossip traffic:
  - **External**: between the (backend) application and Redis.
  - **Internal**: between Redis processes themselves — primary-to-replica replication, and inter-node traffic in a Cluster deployment.
- **Client-facing TLS needs a set of certificate/key files**, all referenced from `redis.conf`:

  ```
  tls-cert-file /path/to/redis.crt
  tls-key-file /path/to/redis.key
  tls-ca-cert-file /path/to/ca.crt
  tls-dh-params-file /path/to/redis.dh
  ```

  - The **CA certificate** is what lets each side verify the other's certificate was issued by a trusted authority.
  - The **server certificate and private key** are what Redis presents to authenticate itself to connecting clients and to perform the actual encryption/decryption.
  - The **Diffie-Hellman parameters file** supports the key-exchange step of the TLS handshake.
- **Enabling `tls-port` does not disable the plaintext port** — both can be listening simultaneously unless you explicitly turn the plaintext one off:

  ```
  port 0
  tls-port 6379
  ```

  Leaving `port` non-zero alongside `tls-port` means an attacker (or a misconfigured client) can simply connect on the plaintext port and bypass TLS entirely — if the goal is "all traffic must be encrypted," `port 0` is not optional, it's the setting that actually enforces that goal.
- **Internal traffic (replication, Cluster) reuses the same certificate configuration, activated by its own separate flags:**

  ```
  tls-replication yes   -- primary-to-replica traffic (Sentinel reuses this same setting)
  tls-cluster yes        -- inter-node traffic in a clustered deployment
  ```

  Forgetting one of these while enabling client-facing TLS is a common gap — the external connection looks secured, but internal replication/cluster traffic is silently still plaintext on the same deployment.
- **TLS has a real, measurable performance cost** (encryption/decryption overhead on every connection) — factor this into capacity planning once TLS is enabled everywhere it's needed, rather than treating it as a free security upgrade.

## Mutual TLS (mTLS): Certificate-Based Client Authentication

- **mTLS lets a client authenticate to Redis using its own certificate instead of a username/password** — the trust relationship runs both directions (Redis verifies the client's certificate, the client verifies Redis's), rather than only the client verifying the server the way ordinary one-way TLS works.
- **mTLS is automatically active as soon as TLS directives are enabled** — this is opt-out, not opt-in, which is easy to miss. If certificate-based client authentication isn't actually wanted (e.g. authentication is meant to stay password/ACL-based, per `database-redis-access-control`), explicitly disable it:

  ```
  tls-auth-clients no
  ```

- **Choose mTLS over password/ACL authentication when clients already have a certificate-issuance/rotation infrastructure in place** (e.g. a service mesh, an internal CA) — it removes password management from the picture entirely for those clients. Where that infrastructure doesn't already exist, standard ACL-based username/password authentication (see `database-redis-access-control`) is usually the more practical starting point.

## Code Example

```python
from redis import asyncio as aioredis

# Client-side TLS configuration — must match what the server actually enforces
# (tls-ca-cert-file, tls-cert-file/tls-key-file for mTLS if tls-auth-clients is on).
client = aioredis.Redis(
    host="internal-redis.example.net",
    port=6379,
    ssl=True,
    ssl_ca_certs="/path/to/ca.crt",
    ssl_certfile="/path/to/client.crt",  # only needed if mTLS (tls-auth-clients) is active
    ssl_keyfile="/path/to/client.key",
)
```
