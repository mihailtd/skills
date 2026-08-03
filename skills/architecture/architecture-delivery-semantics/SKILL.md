---
name: architecture-delivery-semantics

description: Instructs the AI assistant on the three message/event delivery semantics (at-most-once, at-least-once, effectively-exactly-once) as a property of whether a consumer acknowledges before or after processing — generalized across any broker or task queue, not Kafka-specific — the chapter's sharpest insight, that "exactly-once" is never a primitive guarantee, it's at-least-once delivery plus idempotent consumer-side deduplication, which breaks the moment any upstream hop in a multi-stage pipeline lacks that same deduplication — and the concrete capacity math for why a durable queue only buys fault tolerance (absorbing an outage or traffic surge) when the consumer has real throughput headroom over the producer's steady-state rate.
---

# Delivery Semantics Instructions

When supporting the design of a message queue, event stream, or task-queue
consumer — Redis Streams, Celery/RQ with any broker, SQS, or anything with
the same shape — use this tool to reason about delivery semantics as a
direct consequence of *when* a consumer acknowledges work relative to
processing it, not as a fixed property of whichever broker was chosen.

---

## Purpose

This tool helps the AI assistant by:

- explaining that a consumer's delivery guarantee is determined by one
  concrete design decision — does it acknowledge/commit *before* processing
  a message, or *after*? — not by which broker or queue library is in use,
- naming the three semantics precisely: **at-most-once** (ack before
  processing — a crash mid-processing loses the message, but never
  duplicates it), **at-least-once** (ack after processing — a crash before
  the ack causes redelivery and possible duplicate processing, but never
  loses the message), and **effectively-exactly-once** (not a distinct
  primitive — it's at-least-once delivery combined with idempotent
  processing on the consumer side, so duplicates are delivered but have no
  duplicate effect),
- correcting the common misreading of a broker's "exactly-once" feature (a
  transactional producer, a consumer-group "exactly one consumer" guarantee)
  as a standalone guarantee — it always reduces to at-least-once plus
  deduplication somewhere, and it's worth knowing exactly where,
- generalizing "restart from where you left off" vs. "restart from now" as
  a consumer-restart design question that applies to any queue consumer,
  independent of which broker is involved,
- insisting that an end-to-end pipeline's real guarantee is only as strong
  as its weakest hop — a downstream stage using exactly-once machinery gains
  nothing if the event that triggered it already arrived at-least-once from
  an upstream source with no deduplication,
- explaining the concrete fault-tolerance payoff a durable, at-least-once
  queue buys — absorbing a consumer outage or a traffic surge without data
  loss — and the capacity math that determines whether a consumer actually
  recovers from a backlog in bounded time, or falls permanently behind.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- picks ack-before-processing (at-most-once) or ack-after-processing
  (at-least-once) deliberately, based on which failure mode — losing a
  message or possibly duplicating it — is actually worse for the specific
  operation in question, rather than defaulting to whichever the
  broker/library does out of the box,
- treats "exactly-once" claims from any broker or library with informed
  skepticism — asks specifically what mechanism backs the claim, and
  whether it actually requires the consumer's own logic to be idempotent to
  hold in practice,
- connects "at-least-once + idempotent processing" directly to
  **architecture-idempotency-and-at-least-once-delivery** and
  **database-postgresql-idempotency-keys** — this is the same building
  block, not a new one, applied to a queue/stream consumer instead of an
  HTTP endpoint,
- checks every hop of a multi-stage event pipeline for this same property,
  rather than assuming a strong guarantee at one stage protects the whole
  pipeline,
- treats "replay from the last checkpoint" (risking duplicates) vs. "skip
  to the newest message" (risking gaps) as a deliberate per-consumer
  decision tied to whether that consumer's business logic can tolerate
  reprocessing old data or genuinely needs every event,
- provisions consumer capacity with real headroom above the producer's
  steady-state rate, specifically so a buffered outage or surge resolves as
  a bounded, temporary delay instead of a backlog that never shrinks.

---

## Instructions for the AI

1. **Derive the delivery semantic from ack timing, not from the broker**
   - **Ack/commit before processing → at-most-once.** If the consumer
     crashes after acknowledging but before finishing the work, that
     message is gone — the broker already considers it delivered and won't
     redeliver it. No duplicates are possible, but data loss is.
   - **Ack/commit after processing → at-least-once.** If the consumer
     crashes after finishing the work but before acknowledging, the broker
     redelivers the message (to this consumer on restart, or to another
     consumer in the same group) — the work runs again. No message is ever
     silently lost, but duplicate processing is possible.
   - This is true regardless of which specific technology is involved —
     Redis Streams (`XREADGROUP` then `XACK`), a Celery/RQ task queue
     acknowledging a job, an SQS consumer deleting a message, or a Kafka
     consumer committing an offset all reduce to this same before/after
     choice. Identify which side of that choice a given consumer is on
     before reasoning about its correctness.

2. **Choose the semantic based on which failure mode actually costs more**
   - For an operation where a duplicate is cheap and a loss is expensive (an
     alert that must fire, a payment that must eventually be captured),
     at-least-once is the right default.
   - For an operation where reprocessing has a real cost and losing an
     occasional message is tolerable (a low-stakes analytics event, a
     best-effort notification), at-most-once may be an acceptable,
     deliberate choice — but state that it's deliberate, not an accident of
     leaving a default in place.
   - Don't let the framework's default silently make this decision — know
     which one a given consumer actually implements.

3. **Treat "exactly-once" as a composed property, not a primitive**
   - There is no such thing as delivery that's inherently exactly-once end
     to end. What brokers market as "exactly-once" is always at-least-once
     delivery (a message can still be redelivered on a crash) combined with
     a deduplication mechanism — a transactional write, a consumer-group
     mechanism that treats a redelivered-but-already-processed message as a
     no-op — that hides the duplicate from the logic that matters.
   - When evaluating an "exactly-once" claim, ask specifically: what does
     the mechanism actually deduplicate, and at what boundary? A producer
     transaction that prevents *the producer itself* from double-writing
     doesn't protect against a duplicate that originated further upstream,
     before that producer ever saw the event — see point 5.
   - Achieving this effect yourself, without relying on a broker feature, is
     the exact same idempotency-key pattern already covered in
     **architecture-idempotency-and-at-least-once-delivery** and
     **database-postgresql-idempotency-keys** — give each message a stable
     unique identifier, and check-and-record it atomically before acting on
     it. A queue consumer doing this is applying the identical pattern as
     an HTTP endpoint deduplicating retried requests.

4. **Make the consumer-restart replay decision deliberately**
   - On restart after a crash or redeployment, a consumer that doesn't know
     exactly which messages it already finished has two choices: resume
     from the last known checkpoint (reprocessing everything since then —
     more duplicates, no gaps) or skip ahead to the newest available message
     (no duplicates, but anything produced during the downtime is never
     seen — a gap).
   - Pick based on whether the consumer's logic can tolerate reprocessing
     old messages (favor resuming from the checkpoint) or whether stale data
     has no value and only current data matters (favor skipping ahead) — a
     payment processor almost always needs the former; a live dashboard
     alerting on very recent activity may reasonably prefer the latter.

5. **Check every hop of a multi-stage pipeline, not just one**
   - An event pipeline with multiple stages (a trigger, a producer, one or
     more consumers that themselves produce further events) is only as
     strong, end to end, as its weakest hop. A downstream stage using a
     broker's exactly-once machinery gains nothing if the event that
     triggered it arrived at that producer via an at-least-once path with
     no deduplication — from that producer's point of view, two genuinely
     duplicate triggers just look like two independent, both-legitimate
     requests, and its own exactly-once machinery faithfully processes both.
   - When asked to make a pipeline "exactly-once," identify every hop where
     an event crosses a network boundary, and verify deduplication is
     actually present at each one — not just at the hop that happens to
     advertise the strongest guarantee.

6. **Size consumer capacity with real headroom, so buffering actually pays off**
   - The concrete fault-tolerance payoff of a durable, at-least-once queue is
     that it absorbs a consumer outage or a traffic surge instead of losing
     data: a producer keeps publishing at its normal rate while the
     consumer is down or overloaded, the broker buffers what accumulates,
     and the consumer drains the backlog once it's healthy again — but only
     if it was set up with resume-from-checkpoint semantics (point 4), not
     skip-to-latest, which would just discard the backlog on restart.
   - Whether "drains the backlog" actually happens in a reasonable time
     depends on one number: how much *headroom* the consumer's throughput
     has over the producer's steady-state rate. While catching up, the
     consumer has to handle the backlog *and* the ongoing new arrivals at
     the same time, so the backlog only shrinks at the *difference* between
     consumer capacity and producer rate, not at the consumer's full
     capacity. Concretely: an outage of length `T` at producer rate `P`
     leaves a backlog of `T × P` messages; if the consumer can sustain
     capacity `C`, it drains that backlog at a net rate of `(C − P)`, so
     recovery takes roughly `(T × P) / (C − P)`. As `C` approaches `P`,
     that recovery time grows without bound — a consumer with little or no
     spare capacity over its normal load never actually catches up.
   - The practical consequence: don't size a consumer to just barely meet
     its steady-state load. Provision real headroom above the producer's
     typical rate specifically so that an outage or a surge is a temporary,
     bounded delay rather than a permanently growing backlog — the queue
     buys recovery time only if there's spare capacity to spend recovering.

---

## AI decision guidance

When generating delivery-semantics guidance, keep these principles in mind:

- **The semantic comes from ack-before vs. ack-after processing** — this is
  a design decision the consumer's own code makes, not an inherent property
  of the broker.
- **"Exactly-once" is always at-least-once plus deduplication somewhere** —
  find out where, and verify it's actually there, before trusting the
  label.
- **The idempotency-key pattern is the same building block whether it's
  protecting an HTTP endpoint or a queue consumer** — don't treat queue
  deduplication as a different problem requiring different machinery.
- **A pipeline's real guarantee is its weakest hop's guarantee** — verify
  deduplication at every network boundary the event crosses, not just the
  one stage that advertises exactly-once.
- **A queue only buys fault tolerance if the consumer has real throughput
  headroom** — size for meaningfully more than the producer's steady-state
  rate, or a buffered outage/surge becomes a backlog that never recovers.

---

## Success criteria

A strong response should ensure that it:

- **identifies whether a consumer acks before or after processing**, and
  names the resulting semantic (at-most-once or at-least-once) correctly,
- **picks the semantic deliberately** based on which failure mode is worse
  for the specific operation, rather than accepting a framework default
  unexamined,
- **explains any "exactly-once" claim as at-least-once plus deduplication**,
  and identifies specifically what's being deduplicated and where,
- **connects queue/stream deduplication to the same idempotency-key
  pattern** used for HTTP endpoints and database writes, rather than
  treating it as a separate concern,
- **checks multi-stage pipelines at every hop**, not just the hop with the
  strongest advertised guarantee,
- **provisions consumer capacity with real headroom**, and can work through
  the backlog/recovery-time math when asked whether a given setup can
  actually recover from an outage or surge in a reasonable time.

---

## Example prompts for the AI

- "Should we acknowledge this queue message before or after we process it?"
- "Our broker advertises exactly-once delivery — do we still need to worry
  about duplicates?"
- "Should our consumer resume from where it left off after a restart, or
  just process new messages?"
- "We have a three-stage event pipeline — is it safe from duplicates end to
  end?"
- "Our consumer went down for 10 minutes — how long will it take to catch
  up on the backlog?"

---

## Related guidance

Use this tool alongside:

- architecture-idempotency-and-at-least-once-delivery — the same
  at-least-once-plus-deduplication pattern, from the request/API angle
  rather than the queue/stream angle.
- database-postgresql-idempotency-keys — the concrete atomic
  check-and-record implementation this skill's point 3 relies on.
- database-redis-streams (package `database-redis`) — this repository's
  concrete at-least-once-delivery mechanism (`XREADGROUP`/`XACK`/`XCLAIM`)
  that this skill's vocabulary applies to directly.
- database-redis-queues-pubsub (package `database-redis`) — Redis
  structures with no acknowledgment at all (effectively at-most-once, or
  weaker), and when that's an acceptable tradeoff.
- architecture-simplicity — don't reach for a queue/broker, multiple
  independent topics, or elaborate pipeline architecture before the
  cascading-failure or decoupling problem it solves is an actual, current
  problem.
