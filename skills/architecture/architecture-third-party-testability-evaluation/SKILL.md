---
name: architecture-third-party-testability-evaluation

description: Instructs the AI assistant to evaluate a third-party library's testability before adopting it, not after — whether it ships its own testing utilities (a strong positive signal), whether time-based or stateful internals can be injected or faked instead of forcing sleep()-based tests, and whether a framework provides supported integration-testing infrastructure — treating an untestable library as a legitimate reason to reject it, since limited trust in code you don't own means testing is how you validate your assumptions about it, not an afterthought.
---

# Third-Party Library Testability Evaluation Instructions

When supporting the selection of a third-party library, use this tool to
treat testability as a selection criterion evaluated up front, not a problem
to solve after the library is already integrated — you can't easily change
code you don't own, so how well it can be tested has to be known before you
commit to it, not discovered afterward.

---

## Purpose

This tool helps the AI assistant by:

- treating limited trust as the correct default posture toward any library
  the team didn't write — testing is how that trust gets calibrated, so a
  library that resists testing resists having its behavior actually
  verified,
- treating a library that ships its own dedicated testing utilities as a
  genuine positive quality signal worth weighing in the selection decision,
  not just a convenience,
- specifically checking whether time-based or otherwise stateful internals
  (a cache's eviction clock, a scheduler's notion of "now") can be injected
  or faked, since the alternative — sleeping in tests for real elapsed time —
  makes a test suite slow and some scenarios outright impractical to cover,
- treating "we can't test this library's behavior without hitting real
  delays or real infrastructure" as a legitimate reason to reject it, not
  just an annoyance to accept.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- checks, before adopting a library, whether it (or its ecosystem) ships a
  companion testing library or test-support utilities, and weighs the
  answer as part of the selection decision,
- specifically investigates whether the library exposes a seam for
  controlling any internal notion of time or state — a clock/ticker
  abstraction, an injectable state provider — rather than assuming
  `sleep()`-based tests are the only option,
- rejects a library outright when a behavior that matters can't be
  practically verified (not just tediously verified) because of missing
  testability seams,
- checks, for a framework component, whether it provides supported
  infrastructure for spinning it up in an integration test (test profiles,
  an embedded/ephemeral instance, automatic port assignment) rather than
  assuming the team will hand-roll that infrastructure themselves,
- treats a closed-source or unfamiliar library's lack of inspectable source
  as a reason to raise the required test coverage before trusting it, not
  lower it.

---

## Instructions for the AI

1. **Check for a dedicated testing library or test-support utilities first**
   - Before writing a single test against a candidate library, check whether
     its ecosystem already ships companion testing utilities — a
     test-double publisher, a fake clock, an in-memory implementation
     designed specifically for tests. Their existence is a genuine signal
     the library's authors thought about testability, not just an
     incidental convenience.
   - Their absence isn't automatically disqualifying, but it means the team
     will need to build that infrastructure themselves — factor that
     ongoing cost into the comparison against alternatives that already
     provide it.

2. **Investigate whether time-based or stateful internals can be injected**
   - Any library whose behavior depends on elapsed time (cache eviction,
     scheduled tasks, rate limiting, retry backoff) has some internal notion
     of "now." Check specifically whether that notion is injectable — a
     clock, a ticker, an explicit time source passed at construction —
     before assuming the only way to test it is to actually wait.
   - A library that hides this dependency entirely forces tests into one of
     two bad options: sleep for real time (slow test suites, and some
     scenarios — hours- or days-long eviction windows — become outright
     impractical to cover), or skip testing that behavior altogether. Treat
     either outcome as a real cost of the library, not a testing
     inconvenience to shrug off.
   - This repository already applies the injectable-clock pattern to its own
     code — see **python-datetime-testability** — which is exactly the
     property to look for when a third-party library's behavior depends on
     time: does it let you supply your own clock the same way this
     repository's own code is expected to?
   - The same question applies to other hidden state, not just time — does
     the library let you substitute a fake/in-memory backend for whatever
     it depends on internally, or does everything route through a
     hardcoded, hard-to-intercept implementation?

3. **Reject a library outright when a behavior that matters can't be practically tested**
   - If, after investigating, a library offers no way to control a behavior
     the application genuinely depends on being correct, that's sufficient
     grounds to choose a different library — not a reason to accept reduced
     test coverage on faith. The cost of discovering a library's behavior
     doesn't match assumptions, in production, after it was too painful to
     verify in tests, is usually far higher than the cost of picking a
     different library up front.
   - For a closed-source or unfamiliar library where the internals can't be
     inspected to look for injectable seams, testing becomes the *only*
     mechanism available for validating assumptions about its behavior —
     treat that as a reason to require more test coverage before trusting
     it, not less, since there's no other way to catch a wrong assumption
     before production does.

4. **Evaluate integration-testing support for framework-level components**
   - For a library that functions more like a framework component (a web
     framework, a dependency-injection container, a data-access layer),
     check whether it provides supported infrastructure for starting it in
     an integration-test environment — test-specific configuration profiles,
     an embedded or ephemeral instance, automatic resource allocation (like
     picking a free port) — rather than assuming the team will build that
     scaffolding from scratch.
   - The presence of this infrastructure meaningfully lowers the ongoing
     cost of integration testing everything built on top of the framework,
     not just the framework itself — weigh it accordingly in a framework
     selection decision, not only a single-library one.

---

## AI decision guidance

When generating library-testability guidance, keep these principles in mind:

- **Testability is a selection criterion evaluated before adoption**, not a
  problem to solve after a library is already integrated into the codebase.
- **A library shipping its own testing utilities is a genuine positive
  signal** — weigh it, don't treat it as incidental.
- **An injectable clock/state seam is what separates a fast, complete test
  suite from a slow one riddled with `sleep()` calls and untestable
  scenarios** — check for it explicitly, the same way this repository
  checks for it in its own code (**python-datetime-testability**).
- **"We can't practically test this" is sufficient reason to reject a
  library** — it's not a compromise to quietly accept.

---

## Success criteria

A strong response should ensure that it:

- **checks for a companion testing library** before or during library
  evaluation, and factors its presence or absence into the decision,
- **specifically investigates injectable time/state seams**, connecting to
  this repository's own clock-injection pattern where relevant,
- **is willing to reject a library outright** when a behavior that matters
  can't practically be verified, rather than accepting reduced coverage,
- **checks framework-level integration-testing support** as part of
  evaluating a framework, not just a single library.

---

## Example prompts for the AI

- "We're evaluating a caching library — how do we know if it'll be
  testable?"
- "This library's eviction window is measured in hours — how do we test that
  without an hours-long test?"
- "Should the fact that we can't easily test this library's retry behavior
  be a dealbreaker?"

---

## Related guidance

Use this tool alongside:

- python-datetime-testability (package `python-core`) — this repository's
  own injectable-clock pattern, the property to look for in a third-party
  library's time-dependent behavior.
- python-testing-mocking (package `python-core`) — general fake/mock
  construction once a library's testability seams are identified.
- architecture-third-party-defaults-and-concurrency
- architecture-third-party-dependency-conflicts
- architecture-third-party-library-selection-checklist — the first-stop checklist tying this skill together with the other third-party-library evaluation areas.
