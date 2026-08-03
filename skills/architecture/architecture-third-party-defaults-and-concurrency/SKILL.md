---
name: architecture-third-party-defaults-and-concurrency

description: Instructs the AI assistant that a third-party library's shipped defaults (especially timeouts) and concurrency model become the calling application's problem the moment it's imported — auditing and explicitly setting anything that affects SLA/reliability instead of trusting convention-over-configuration defaults, matching a library's blocking/async model to the calling code's own execution model instead of wrapping around a mismatch, and evaluating stateful/distributed third-party components (schedulers, caches) for multi-node correctness before depending on them in a horizontally-scaled deployment.
---

# Third-Party Defaults and Concurrency Instructions

When supporting the selection or integration of a third-party library, use this
tool to treat the library's defaults and concurrency model as decisions the
calling application inherits, not implementation details that can be left
alone — the library's code becomes the application's code the moment it's
imported, including whatever assumptions its authors baked into its defaults.

---

## Purpose

This tool helps the AI assistant by:

- treating "convention over configuration" as acceptable for prototyping but
  never acceptable unverified in production — a library's shipped defaults
  were chosen for a generic use case, not necessarily this one,
- naming timeouts as the sharpest, most universal example of a default that
  silently determines whether the calling service can meet its own SLA, and
  insisting they be set explicitly rather than inherited,
- matching a library's blocking-vs-asynchronous execution model to the
  calling code's own model, rather than papering over a mismatch with a
  wrapper that relocates the cost instead of removing it,
- evaluating any third-party component with meaningful internal state (a
  scheduler, a distributed cache, anything backed by its own persistence)
  specifically for multi-node correctness before depending on it in a
  horizontally-scaled deployment, since a bug here often stays invisible
  until traffic actually forces a scale-out.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- explicitly reviews and sets every third-party default that affects
  reliability or latency (timeouts above all) before a library reaches
  production, rather than trusting whatever ships as the default,
- calculates a timeout, retry, and backoff policy from the calling service's
  own SLA and the risk of cascading failure in a call chain, not from
  whatever number felt reasonable,
- checks whether a library's API is blocking or asynchronous before calling
  it from code with the opposite model, and treats a mismatch as a real
  design decision (wrap it, own a thread pool, and tune it — or don't use
  the library) rather than something to route around casually,
- prefers a library with a genuinely async-native implementation over a
  blocking one wrapped in a promise/future facade, when performance is a
  real constraint, since the wrapper's thread pool is overhead the native
  version doesn't pay,
- specifically investigates whether a stateful third-party component
  supports partitioning or leader election before trusting it across
  multiple nodes, rather than assuming single-node integration test success
  generalizes.

---

## Instructions for the AI

1. **Audit every default that affects reliability, especially timeouts**
   - Treat an unset timeout as equivalent to "this call can block the caller
     indefinitely, or for whatever the library author guessed was
     reasonable" — some defaults are surprisingly permissive (an infinite
     default timeout is not hypothetical; some standard-library HTTP clients
     ship with exactly that).
   - Derive the actual timeout from the calling service's own SLA and its
     position in a call chain: if the service must respond within 100ms, a
     dependency it calls synchronously must fail (not just respond) well
     inside that budget, leaving room for retries or a fallback.
   - In a chain of service-to-service calls, flag that a high timeout on any
     one link risks cascading failure across the whole chain — a slow
     dependency doesn't just slow down the caller, it can tie up the
     caller's own limited worker threads, degrading everything else that
     caller is supposed to be handling concurrently.
   - Apply the same scrutiny to every other default with a reliability or
     performance consequence — connection pool size, retry counts and
     backoff, buffer sizes — not just timeouts specifically; timeouts are
     the clearest example, not the only one.

2. **Match the library's concurrency model to the calling code's own model**
   - Before calling a third-party method, know whether it's blocking or
     asynchronous, and know the execution model of the code calling it. Code
     running on a shared event loop (an async web framework, a
     single-threaded reactor model) that calls a blocking method stalls that
     entire loop for every concurrent request it's supposed to be handling,
     not just the one that triggered the call.
   - If a genuinely async-native alternative exists, prefer it over a
     blocking library wrapped in an async facade — the async-native version
     doesn't need a dedicated thread pool just to avoid blocking its caller.
   - If wrapping a blocking library is the only option, recognize that the
     wrapper doesn't remove the concurrency cost, it relocates it: a
     dedicated thread pool now needs sizing, monitoring, and tuning, the
     same "cost relocates, it doesn't disappear" pattern covered in
     **architecture-flexibility-complexity-tradeoffs**. Budget for that
     ownership explicitly rather than treating the wrapper as a free fix.
   - The reverse direction (blocking code calling an async library) is
     comparatively simple — block on the result with an explicit timeout —
     but still requires deciding that timeout deliberately, per point 1,
     rather than blocking indefinitely on the underlying future/promise.

3. **Evaluate stateful, distributed-sensitive components before depending on them at scale**
   - A library with meaningful internal state — a task scheduler, an
     embedded cache with its own persistence, anything that coordinates
     work across calls — behaves very differently at one node versus many.
     A single-node integration test passing is not evidence it's safe to
     run on N nodes.
   - Specifically ask: if this component runs on multiple nodes
     simultaneously, can the same unit of work (the same scheduled job, the
     same cache entry) be processed more than once, or processed
     inconsistently across nodes? Does the library have documented support
     for partitioning work across nodes, or a leader-election mechanism to
     ensure only one node acts at a time?
   - Treat the absence of a clear answer as a red flag serious enough to
     revisit the library choice, not a detail to defer — this class of bug
     often only surfaces under a genuine multi-node deployment, frequently
     during exactly the traffic surge (a launch, a holiday) when the
     business least wants an outage.

---

## AI decision guidance

When generating third-party integration guidance, keep these principles in
mind:

- **A library's defaults are a decision inherited by the calling
  application** — audit and set them explicitly, especially anything
  touching timeouts, before production use.
- **Concurrency-model mismatches must be resolved deliberately** — matching
  or wrapping — never routed around casually; a wrapper relocates cost, it
  doesn't remove it.
- **Prefer async-native libraries over blocking libraries wrapped in a
  facade** when performance genuinely matters, since the wrapper's thread
  pool is pure overhead the native version avoids.
- **Stateful components need explicit multi-node evaluation** before being
  trusted in a horizontally-scaled deployment — single-node test success
  proves nothing about multi-node correctness.

---

## Success criteria

A strong response should ensure that it:

- **identifies and sets the relevant timeout/retry/backoff defaults**
  explicitly, tied to the calling service's own SLA,
- **names the library's concurrency model and the calling code's own**, and
  resolves any mismatch deliberately rather than working around it silently,
- **calls out the ongoing cost of wrapping a blocking library** (thread pool
  ownership) rather than presenting the wrapper as a free fix,
- **raises multi-node correctness explicitly** for any stateful third-party
  component before it's trusted in a scaled-out deployment.

---

## Example prompts for the AI

- "We just added this HTTP client library — what settings should we not
  leave at their defaults?"
- "Is it safe to call this library from our async request handler?"
- "We're adding a job-scheduling library — what do we need to check before
  running it on more than one instance?"

---

## Related guidance

Use this tool alongside:

- architecture-flexibility-complexity-tradeoffs — the general "cost
  relocates, it doesn't disappear" principle behind wrapping a blocking
  library in an async facade.
- architecture-code-duplication-tradeoffs — the microservice-vs-library
  extraction tradeoff, relevant when a third-party dependency's scalability
  limits push toward extracting a dedicated service instead.
- architecture-third-party-testability-evaluation
- architecture-third-party-dependency-conflicts
- architecture-third-party-library-selection-checklist — the first-stop checklist tying this skill together with the other third-party-library evaluation areas.
- architecture-idempotency-and-at-least-once-delivery — the same "any network call can fail ambiguously" reasoning, applied to designing retry-safe operations rather than configuring a specific call's timeout.
