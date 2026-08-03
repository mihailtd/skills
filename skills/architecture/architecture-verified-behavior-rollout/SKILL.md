---
name: architecture-verified-behavior-rollout

description: Instructs the AI assistant on the dark-mode/light-mode rollout technique for refactors that swap an implementation and need strong behavioral-equivalence verification — running old and new implementations side by side behind a dispatcher, comparing their outputs while returning the old result (dark mode) then the new one (light mode), widening exposure as discrepancies quiet down, and only then removing the old path — plus the cost tradeoff specific to single-threaded-per-worker runtimes (doubled work per call) and the sampled-comparison mitigation for performance-sensitive or high-traffic candidates.
---

# Verified Behavior Rollout Instructions

When supporting the rollout of a refactor that swaps one implementation for
another and needs strong proof that behavior hasn't changed, use this tool
to recommend running both implementations side by side and comparing their
outputs before ever fully committing to the new one — rather than relying
on pre-deployment testing alone to catch every discrepancy.

---

## Purpose

This tool helps the AI assistant by:

- recognizing the specific difficulty of verifying a refactor's success:
  proving the *absence* of a behavior change is harder than detecting a
  behavior change, and testing alone often isn't enough to be confident
  at scale,
- providing a concrete technique — running the old and new
  implementations side by side, comparing their outputs, and only
  gradually cutting over — for building that confidence directly from
  production traffic rather than test cases alone,
- staging the cutover in two distinct phases (returning the old result
  while comparing, then returning the new result while still comparing)
  so behavior never changes for real users until confidence is already
  established,
- naming the real cost of this technique — roughly doubled work per call
  — and recommending a sampled-comparison mitigation for performance
  sensitive or high-traffic candidates, with particular attention to
  runtimes where that doubled cost is serial rather than free background
  parallelism.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- recommends this technique specifically for candidates that swap an
  implementation and need strong behavioral-equivalence verification —
  not for every refactor, since the added complexity of running two
  implementations side by side isn't free,
- stages the rollout through a genuine dark-mode phase (old result
  returned, new result computed and compared silently) before ever
  entering a light-mode phase (new result returned, comparison continues
  as a safety net against unrelated concurrent changes),
- widens exposure gradually as discrepancies quiet down, rather than
  flipping from 0% to 100% once tests pass,
- removes the old implementation, the comparison logic, and the dispatcher
  once the new path has been stable at full exposure for an adequate soak
  period — treating this cleanup as part of the technique, not optional,
- accounts explicitly for the technique's real cost — doubled work per
  call — and recommends sampling the comparison rather than running it on
  every request when that cost is a genuine concern.

---

## Instructions for the AI

1. **Recognize when this technique is warranted**
   - Recommend it specifically for a refactor that swaps an
     implementation (a new algorithm, a new library, a rewritten code
     path) and needs strong proof that its behavior matches the old one —
     the case where "did this change anything?" is hard to answer from
     tests alone, because a successful refactor is invisible to users and
     proving an absence of change is inherently harder than spotting a
     presence of one.
   - Don't recommend it reflexively for every refactor — the added
     complexity (a temporary dispatcher, dual execution, comparison
     logging) is a real cost that should be reserved for candidates where
     the verification need actually justifies it.

2. **Implement the new logic alongside the old, behind a dispatcher**
   - Implement the refactored logic in a separate path from the current
     logic, and turn the original entry point into a small dispatcher
     capable of calling either implementation.

3. **Dark mode: verify silently before changing what callers see**
   - Call both implementations on every invocation, log any discrepancy
     between their outputs, but return the *old* result — behavior is
     completely unchanged from the caller's perspective while evidence
     accumulates.
   - Fix discrepancies as they surface, and widen exposure (a larger
     traffic percentage, more environments) as confidence grows. Don't
     rush straight to full exposure — the point of this phase is to find
     problems while they're still invisible to real behavior.

4. **Light mode: cut over while still verifying**
   - Once dark mode is stable, switch to returning the *new* result while
     continuing the same dual-call-and-compare — this also guards against
     a concurrent, unrelated change landing in the old path that the new
     one doesn't mirror, which dark mode alone wouldn't catch.
   - Widen to full exposure the same gradual way as dark mode.

5. **Remove everything once stable — this is part of the technique, not optional**
   - Once stable at full exposure for an adequate soak period, remove the
     old implementation, the comparison logic, and the dispatcher
     entirely. Only the new implementation should remain where the old
     one once was.
   - Leaving any of this in place after cutover is confirmed stable
     reintroduces exactly the kind of stale transitional-artifact problem
     a completed refactor should avoid.

6. **Account for the real cost, and mitigate it with sampling when needed**
   - Dual execution roughly doubles the work done per call — extra
     network/database round trips, extra CPU. In a single-threaded-per-
     worker runtime, that added cost is serial, not free background
     parallelism, which makes it a real latency concern, not just an
     abstract inefficiency.
   - For a performance-sensitive or high-traffic candidate, recommend
     comparing at a **sampled rate** (start low — e.g. 5% — and ramp up as
     discrepancies quiet down) rather than comparing on 100% of calls.
     This controls both the added latency and the load on whatever's
     logging the comparisons, which can itself become a bottleneck at
     full volume if comparisons are frequent and voluminous.

---

## AI decision guidance

When generating verified-rollout guidance, keep these principles in mind:

- **Reserve this technique for genuine implementation swaps needing strong
  behavioral proof** — not a default for every refactor.
- **Dark mode must come before light mode** — never let a caller-visible
  result change before confidence has been established silently.
- **Widen exposure gradually in both phases** — the point is finding
  problems before they matter, not rushing to completion.
- **Removal of the dispatcher/comparison logic is part of the technique**
  — a "temporary" verification setup left in place indefinitely is a
  failure mode, not a safe steady state.
- **Name the real cost (doubled work per call) and mitigate with sampling**
  when performance or traffic volume makes full-rate comparison risky.

---

## Success criteria

A strong response should ensure that it:

- **recommends this technique only where implementation-swap verification
  genuinely warrants it**, not reflexively,
- **stages dark mode before light mode**, with caller-visible behavior
  unchanged throughout dark mode,
- **widens exposure gradually** rather than jumping to full traffic,
- **includes removal of the dispatcher and comparison logic** as an
  explicit final step, not an afterthought,
- **names the doubled-work cost and recommends sampling** for
  performance-sensitive or high-traffic cases.

---

## Example prompts for the AI

- "We're replacing this core algorithm — how do we verify nothing broke
  before fully switching over?"
- "Is it safe to run both the old and new implementations at the same
  time in production?"
- "This code path is very high-traffic — running a full comparison on
  every request feels risky. What should we do instead?"
- "We finished the migration and it's been stable for weeks — what's left
  to clean up?"

---

## Related guidance

Use this tool alongside:

- python-dark-mode-light-mode-rollout (package `python-core`) — the
  concrete Python implementation of this technique, including the
  dispatcher, comparison logging, and sampled-rate mechanics.
- architecture-refactoring-plan-structure — the mandatory cleanup
  milestone this technique's final removal step feeds directly into.
- architecture-refactoring-scope-classification — this technique is
  typically warranted for At-Scale candidates specifically; a small,
  contained Local refactor rarely needs this level of verification
  machinery.
- architecture-data-storage-schema-evolution — the sibling expand-contract
  pattern for storage schema changes specifically, where the "old and new
  coexist, then cut over, then remove the old" shape is the same idea
  applied to data rather than code paths.
