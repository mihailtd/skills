---
name: database-postgresql-idempotency-keys

description: Guides implementing request deduplication (idempotency keys) in PostgreSQL as a single atomic INSERT ... ON CONFLICT DO NOTHING RETURNING statement — never a separate SELECT-then-INSERT, which is a genuine race condition under concurrent requests even against one PostgreSQL instance, not just across multiple service nodes — plus keeping the key-insert and the associated business write in the same transaction, and the remaining gap this pattern doesn't close (an external side effect, like an actual email send or payment API call, that can't be wrapped in the same transaction).
---

# PostgreSQL Idempotency Keys for Request Deduplication

When a client (or a queue, or a retrying caller) can send the same logical
request more than once — because it never received a response the first
time, not because it actually wants to repeat the action — the receiving
service needs to detect and ignore the duplicate. This is the concrete,
PostgreSQL-native implementation of the pattern covered conceptually in
**architecture-idempotency-and-at-least-once-delivery**.

## 1. One Atomic Statement, Never Check-Then-Insert

The mistake to avoid: checking whether an idempotency key exists with a
`SELECT`, and inserting it with a separate `INSERT` if it doesn't. Two
concurrent requests carrying the same key can both run the `SELECT` before
either runs the `INSERT` — both see "not present" and both proceed. **This
race happens with two concurrent requests against a single PostgreSQL
instance** — it has nothing to do with running multiple service nodes; more
nodes just means more opportunities for it to occur.

The fix is a single statement that checks and records the key atomically,
using `ON CONFLICT` and `RETURNING` together:

```sql
CREATE TABLE idempotency_keys (
    key TEXT PRIMARY KEY,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO idempotency_keys (key)
VALUES ($1)
ON CONFLICT (key) DO NOTHING
RETURNING key;
```

- If the key wasn't already present, the row is inserted and `RETURNING`
  yields one row — this is a new request; proceed with the operation.
- If the key was already present, `ON CONFLICT (key) DO NOTHING` fires and
  `RETURNING` yields **zero rows** — this is a duplicate; skip the operation
  entirely and, if a response is still owed to the caller, return the
  previously-recorded outcome rather than silently doing nothing (see
  section 3).

There is no window between "check" and "record" for a second request to
interleave — the database itself guarantees this is one atomic operation.
For the underlying `ON CONFLICT` syntax in more depth, see
**database-postgresql-merge-upsert-synchronization** — this skill is
specifically about the request-deduplication use of that mechanism, not
`ON CONFLICT`'s full syntax.

## 2. Keep the Key Insert and the Business Write in the Same Transaction

Recording the idempotency key atomically solves the *concurrent duplicate
request* race. It does not, by itself, solve a second problem: what happens
if the operation's own side effect fails *after* the key was recorded? A
naive implementation that records the key first and performs the write
second risks marking a request as "processed" when it actually failed —
any genuine retry is then wrongly treated as a duplicate and silently
dropped.

For a side effect that's itself a database write (inserting an order, a
payment record, a ledger entry), put the key insert and the business write
in the **same transaction**:

```sql
BEGIN;

INSERT INTO idempotency_keys (key)
VALUES ($1)
ON CONFLICT (key) DO NOTHING
RETURNING key;
-- application checks: 0 rows → duplicate, ROLLBACK and return early

INSERT INTO orders (id, customer_id, total)
VALUES ($2, $3, $4);

COMMIT;
```

If the row-count check shows the key was a duplicate, roll back immediately
without touching `orders` at all. If the key insert succeeds, the business
write happens in the same transaction — so if the business write fails for
any reason, the whole transaction (including the key insert) rolls back
together, and a genuine retry will correctly find the key absent and be
allowed to proceed.

## 3. Store the Outcome, Not Just the Key, When the Caller Needs a Response

A caller that retries usually still needs *a* response — not just to be
silently ignored on the second attempt. Store enough of the operation's
result alongside the key to return it again on a detected duplicate, rather
than returning an empty or generic response that leaves the caller unsure
whether anything happened:

```sql
CREATE TABLE idempotency_keys (
    key TEXT PRIMARY KEY,
    response_body JSONB,
    status_code INT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

On a duplicate (zero rows from the `RETURNING` clause), look up the stored
`response_body`/`status_code` for that key and return exactly what the
original, successful attempt returned.

## 4. What This Pattern Doesn't Solve: External Side Effects

This pattern fully closes the gap for side effects that are themselves
PostgreSQL writes, because the key insert and the write can share one
transaction. It does **not** fully close the gap for a side effect that
happens *outside* the database — an actual outbound email send, a call to a
payment provider's API — because that external call can't be wrapped inside
a PostgreSQL transaction the way a second `INSERT` can.

Recording the key, then making the external call, then marking success still
leaves a window where the process can crash (or the network can fail)
between "external call succeeded" and "outcome recorded" — the same
partial-failure problem this pattern was built to prevent, just moved one
layer out. Solving that fully needs a different, heavier pattern (an outbox
table plus a separate dispatcher, or a saga/compensating-action approach) —
treat that as a deliberate escalation for when an external side effect's
duplicate cost genuinely justifies it, not something to reach for by
default. For most services, narrowing the window as far as possible (record
the key, make the call immediately after, keep that gap as small as
practical) is a reasonable, proportionate tradeoff — see
**architecture-simplicity** for the general principle behind not
building the heavier version until it's actually needed.

## Related guidance

- **architecture-idempotency-and-at-least-once-delivery** (package
  `architecture`) — the conceptual framing this skill implements: at-least-once
  delivery, why naive retries duplicate side effects, and why this pattern
  works unchanged whether the service is single-instance or scaled out.
- **database-postgresql-merge-upsert-synchronization** — the general
  `ON CONFLICT`/`MERGE` syntax this skill's pattern is one specific
  application of.
- **database-postgresql-timestamp-timezone-management** — `TIMESTAMPTZ` for
  the `created_at` column, consistent with this package's general timestamp
  guidance.
