---
name: architecture-idempotency-and-at-least-once-delivery

description: Instructs the AI assistant that any network call — even from a single-instance, non-"distributed" service calling one external API or database — can fail after its side effect already happened, so a naive retry gives at-least-once (not exactly-once) delivery; distinguishes naturally idempotent operations (GET, DELETE) from ones that need deliberate redesign (sending an email, charging a card); and explains why the fix (an idempotency key checked and recorded atomically) is the same building block whether the service is a single instance or scaled to many, so it doesn't require adopting distributed-systems architecture up front to use correctly.
---

# Idempotency and At-Least-Once Delivery Instructions

When supporting the design of any operation that can be retried — a client
retrying a failed request, a queue redelivering a message, a caller timing out
and trying again — use this tool to reason about at-least-once delivery and
idempotency. This applies the moment a service makes *any* network call, not
only once an application has become a "real" distributed system with multiple
nodes — a single-instance service calling one external API already has this
problem.

---

## Purpose

This tool helps the AI assistant by:

- establishing that a network call's failure is ambiguous — the callee may
  have completed its side effect before the failure occurred, so the caller
  genuinely cannot tell "definitely didn't happen" from "definitely
  happened, but I didn't hear back," and treating these as the same case is
  a bug,
- explaining that a naive retry policy gives **at-least-once** delivery, not
  exactly-once — an operation retried enough times will eventually succeed,
  but may also execute more than once,
- distinguishing operations that are naturally safe to retry (idempotent by
  nature — a lookup, a delete) from ones that aren't (sending an email,
  charging a card) and need deliberate redesign or explicit deduplication to
  be made retry-safe,
- correcting the assumption that this is exclusively "distributed systems"
  territory reserved for multi-node architectures — the same race condition
  this tool addresses (a naive check-then-act sequence) happens with plain
  concurrent requests against a single instance and a single database, no
  horizontal scaling required, and the fix that's correct there is *also*
  what makes the logic safe once a service does scale out, without needing
  to change anything.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- treats a timeout or connection failure from any external call (another
  service, a payment provider, a database) as "unknown outcome," never as
  "definitely didn't happen" — and designs the retry behavior accordingly,
- identifies whether an operation is naturally idempotent (safe to retry as-is)
  or not, and for non-idempotent operations, either redesigns the operation
  to be idempotent or adds explicit deduplication — doesn't just retry and
  hope,
- when redesigning an operation to be idempotent by sending full state
  instead of incremental deltas, also accounts for the ordering problem that
  introduces (a late-arriving retry must not overwrite a newer state) and
  scopes ordering guarantees to the right key (e.g. per-entity, not global),
- implements deduplication as a single atomic "check and record" operation,
  never a separate check-then-act sequence, since the latter is a race
  condition regardless of whether the service is single-instance or scaled
  out,
- doesn't reach for heavier distributed-systems patterns (CQRS, event
  sourcing, multi-node coordination) before they're actually needed — this
  tool's core pattern (an idempotency key, checked and recorded atomically)
  is deliberately the minimal building block that works correctly at both
  ends of that spectrum without modification.

---

## Instructions for the AI

1. **Treat any network call's failure as an unknown outcome, not a known one**
   - A service calling another service, a payment provider, or even its own
     database over the network cannot distinguish "the operation never
     started" from "the operation completed, but the success response was
     lost" — a timeout means exactly that: no information was received in
     time, not that nothing happened.
   - This is true the moment any network call exists in the system — a
     single-instance service calling one external HTTP API is already
     operating under these rules. Don't reserve this reasoning for
     architectures with multiple nodes or services; it starts at the first
     network hop.

2. **Recognize that retries give at-least-once delivery, not exactly-once**
   - If a caller retries a failed call until it gets a success response, the
     underlying operation is guaranteed to eventually execute — but it may
     execute more than once, since an earlier attempt's success might have
     been the one whose response got lost. Retrying without addressing this
     converts "might not happen" into "might happen more than once," which
     is progress, but not correctness, for anything where duplicate
     execution has a real consequence.
   - Name the consequence concretely for the operation in question — a
     duplicate email is a minor annoyance; a duplicate payment charge is a
     financial and trust-damaging bug. The acceptable handling differs by
     how expensive a duplicate actually is, but "we retry, so we're safe"
     is never true on its own.

3. **Distinguish naturally idempotent operations from ones that need redesign**
   - Read/lookup operations and deletes are naturally idempotent — retrying
     a "get" or a "delete this ID" any number of times produces the same
     end state as doing it once (assuming nothing else concurrently changes
     the data). These are safe to retry with no additional design work.
   - Operations that create a new side effect each time they run (sending an
     email, charging a card, incrementing a counter, appending an event)
     are not idempotent by default. Retrying them naively duplicates the
     side effect.
   - Some of these can be redesigned to become idempotent: sending the
     *complete current state* of an entity (a full snapshot) rather than an
     *incremental change* (a delta) means a duplicate send just re-asserts
     the same state instead of compounding it (compare "cart now contains 2
     of item A" — safe to repeat — against "add 1 of item A to the cart" —
     unsafe to repeat). This redesign has a real cost (more data transferred
     per message, more to serialize) and introduces a new problem covered
     in point 4 — it's a deliberate tradeoff, not a free win.
   - Operations that fundamentally can't be made idempotent this way (an
     actual charge to a payment network, an actual email dispatch) need
     explicit deduplication instead — see point 5.

4. **When redesigning toward full-state messages, solve ordering explicitly**
   - A full-state redesign only stays correct if messages are applied in
     the order they were produced — a retried, now-stale message that
     arrives after a newer one must not be allowed to overwrite it. Getting
     this wrong reintroduces the exact inconsistency the redesign was
     meant to fix, just via a different mechanism (out-of-order delivery
     instead of duplicate accumulation).
   - Scope the ordering guarantee to the right key — usually per-entity
     (per user, per order, per cart), not globally across everything the
     system produces. A global total order is expensive and rarely
     necessary; an entity only needs its own messages to arrive in the
     order they were produced relative to each other.

5. **Implement deduplication as a single atomic operation, never check-then-act**
   - The correct general-purpose fix for a genuinely non-idempotent
     operation: give every request/message a unique identifier (an
     idempotency key), and before executing the operation, atomically check
     whether that key has been seen before *and* record it in the same
     operation.
   - **Never split this into two steps** — "check if the key exists" as one
     call, then "insert the key" as a separate call, with the actual
     operation in between. Two concurrent requests carrying the same key can
     both pass the check before either has recorded it, and both proceed —
     this is a genuine race condition, and it happens with two concurrent
     requests hitting one instance and one database, not only with multiple
     service instances behind a load balancer. Concurrency causes it;
     horizontal scale just adds more opportunities for it to occur.
   - The fix is a single atomic "insert this key if absent, and tell me
     whether it was actually inserted" operation — an upsert-style database
     primitive that both checks and records the key in one round trip, with
     no window for another request to interleave. See
     **database-postgresql-idempotency-keys** for the concrete
     `INSERT ... ON CONFLICT` pattern that implements this correctly.
   - This is the reason the pattern doesn't require "building a distributed
     system" up front: the atomic-upsert-based version is already correct
     whether the service is one instance or many, because the database's
     atomicity guarantee — not any coordination logic the application would
     otherwise have to write — is what prevents the race. Scaling out later
     doesn't require revisiting this logic at all.

---

## AI decision guidance

When generating idempotency/retry guidance, keep these principles in mind:

- **Any network call, even from a single-instance service, can fail with an
  unknown outcome** — this reasoning doesn't wait for a multi-node
  architecture to become relevant.
- **Retries alone give at-least-once, not exactly-once, delivery** — always
  name what a duplicate execution would actually cost before deciding
  whether that's acceptable.
- **Redesigning toward idempotent (full-state) messages trades duplicate-safety
  for an ordering requirement** — don't recommend one without the other.
- **Deduplication must be a single atomic operation** — a separate
  check-then-act sequence is a race condition under concurrency alone, with
  or without multiple nodes.
- **This pattern is the right-sized building block, not a sign of
  over-engineering** — it's correct at one instance and stays correct at N,
  which is exactly why it's worth using even when a system is currently
  simple, rather than something to defer until "we actually need distributed
  systems."

---

## Success criteria

A strong response should ensure that it:

- **treats a call's timeout/failure as an unknown outcome**, not an assumed
  non-occurrence, regardless of the system's current scale,
- **names the real cost of a duplicate execution** for the specific
  operation in question, rather than treating "we retry" as sufficient on
  its own,
- **distinguishes naturally idempotent operations from ones needing
  redesign or explicit deduplication**,
- **pairs any full-state-redesign recommendation with an explicit ordering
  guarantee**, scoped to the right key,
- **implements deduplication as a single atomic check-and-record
  operation**, never a separate check-then-act sequence, and explains why
  that holds regardless of node count.

---

## Example prompts for the AI

- "Our client retries failed requests — is that enough to make our API
  safe?"
- "We send an event every time an item is added to a cart — is that safe to
  retry?"
- "Do we need to worry about this race condition if we're only running one
  instance of this service?"
- "How do we make this payment-processing endpoint safe to retry?"

---

## Related guidance

Use this tool alongside:

- database-postgresql-idempotency-keys (package `database-postgresql`) —
  the concrete atomic `INSERT ... ON CONFLICT` pattern this skill's point 5
  points to.
- architecture-simplicity — the general principle behind not reaching for
  CQRS, event sourcing, or multi-node coordination patterns before they're
  actually needed; this skill's pattern is deliberately the minimal
  building block, not a step toward those.
- architecture-third-party-defaults-and-concurrency — the same
  "every network call can fail" reasoning, applied to third-party library
  timeouts and retries specifically.
- architecture-delivery-semantics — the same at-least-once-plus-deduplication
  pattern applied to a queue/event-stream consumer instead of an HTTP
  request, including why a broker's "exactly-once" feature is never a
  standalone guarantee.
- python-fastapi-partial-updates (package `python-fastapi`) — a concrete
  example of a naturally idempotent operation shape (PATCH touching only
  explicitly-provided fields), safe to retry without the deduplication
  machinery this skill otherwise requires.
