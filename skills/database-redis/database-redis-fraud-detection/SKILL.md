---
name: database-redis-fraud-detection

description: Guides the agent on composing Redis's Bloom filters, time series, and vector similarity search into a real-time fraud/anomaly detection pattern — verifying whether a user's current behavior (location, time, transaction pattern) matches their established norm, with minimal memory overhead per user.
---

# Redis Fraud Detection Patterns

You are an expert in designing real-time fraud/anomaly detection with Redis. This is a synthesis skill: fraud detection isn't a single Redis feature, it's a composition of primitives already covered elsewhere in this package — use this skill to recognize the pattern and route to the right primitive, not as a replacement for `database-redis-probabilistic-data-structures`, `database-redis-time-series`, or `database-redis-vector-similarity-search`.

- **The core technique is behavioral fingerprinting with Bloom filters: record what's "normal" for a given user, then check new activity against that record.** A per-user Bloom filter accumulates the locations, times, or other categorical attributes already seen as legitimate for that user, at a small, roughly fixed memory cost per user regardless of how much history accumulates:

  ```
  BF.MADD usr:1 ITA 11
  BF.MEXISTS usr:1 CAN 3
  ```

  This adds `ITA` and `11` as two independent members of the *same* per-user filter — it records "this user has transacted from Italy before" and "this user has been active at hour 11 before" as two separate facts sharing one filter, not a single compound "Italy-at-11am" fact. `BF.MEXISTS usr:1 CAN 3` then checks each value independently: has `CAN` ever been added, and has `3` ever been added. Either coming back negative is a signal worth flagging. Be deliberate about this trade-off: sharing one filter across attribute types is compact, but only safe when the value domains can't collide (a country code will never look like an hour number) — if you need a true compound check ("has this exact country+hour pair been seen," not each independently), use a filter keyed on the combined value instead (e.g. add the member `"ITA:11"` rather than `"ITA"` and `"11"` separately).
- **Recognize the questions this pattern answers**, and design one Bloom filter (or one filter per attribute) per question:
  - "Has this user transacted from this location before?"
  - "Has this user been active at this time of day before?"
  - "Has this user purchased in this product category before?"
  - "Has this credit card been reported lost/stolen?" (a shared, not per-user, filter)
  - Remember the false-positive/no-false-negative asymmetry from `database-redis-probabilistic-data-structures`: a `BF.MEXISTS` **negative** result (never seen) is certain and safe to act on (require step-up verification); a **positive** result has a small chance of being wrong, so don't treat "seen before" as an absolute guarantee of legitimacy for very high-stakes decisions — combine it with other signals rather than trusting it alone.
- **Layer in time series when the fraud signal is about a pattern over time, not just a single attribute check** — e.g. a sudden spike in transaction frequency or amount for an account, which a point-in-time Bloom filter check can't detect on its own. See `database-redis-time-series` for tracking a metric stream per account and querying/aggregating recent activity to detect anomalous spikes.
- **Layer in vector similarity search when the fraud signal is "does this transaction resemble known fraudulent transactions,"** rather than "has this exact attribute been seen before." This is a genuinely different question — VSS compares a transaction's feature vector against a database of known-fraud vectors for a similarity score, where Bloom filters only ever answer yes/no membership against a user's own history. See `database-redis-vector-similarity-search`.
- **A minimal, real-time fraud check composes cheaply**: a handful of `BF.MADD`/`BF.MEXISTS` calls per transaction, each answering one yes/no question, combined in application logic into a risk score or a binary "require additional verification" decision. This is deliberately lightweight — the point of using probabilistic structures here is to make this check cheap enough to run synchronously, in the transaction path, not as an offline batch job running after the fact.
- **Don't build a custom exact-match equivalent (a Set per user per attribute) unless there's a specific reason the Bloom filter's tiny false-positive rate is unacceptable for this check** — for fraud *screening* (deciding whether to ask for extra verification, not a final legal determination), the memory and speed savings usually far outweigh the marginal accuracy cost. See `database-redis-probabilistic-data-structures` for that general trade-off.

## Code Example

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def record_legitimate_activity(user_id: str, country: str, hour: int) -> None:
    """Call this after a transaction is confirmed legitimate, to grow the user's baseline.
    Records country and hour as independent facts sharing one filter."""
    await client.bf().madd(f"usr:{user_id}", country, str(hour))

async def is_country_and_hour_familiar(user_id: str, country: str, hour: int) -> tuple[bool, bool]:
    """A False here is a certain 'never seen this before' — a strong step-up-auth signal.
    Returns (country_seen_before, hour_seen_before) — independent checks, not a compound one."""
    country_seen, hour_seen = await client.bf().mexists(f"usr:{user_id}", country, str(hour))
    return bool(country_seen), bool(hour_seen)

async def record_legitimate_pair(user_id: str, country: str, hour: int) -> None:
    """For a true compound check, key the member on the combined value instead."""
    await client.bf().add(f"usr:{user_id}:pairs", f"{country}:{hour}")

async def is_country_hour_pair_familiar(user_id: str, country: str, hour: int) -> bool:
    return bool(await client.bf().exists(f"usr:{user_id}:pairs", f"{country}:{hour}"))
```
