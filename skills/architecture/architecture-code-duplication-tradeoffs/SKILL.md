---
name: architecture-code-duplication-tradeoffs

description: Instructs the AI assistant to treat code duplication across independently-owned codebases or teams as a legitimate tradeoff rather than an automatic defect — using Amdahl's law to reason about the synchronization cost that removing it introduces, and choosing deliberately between a shared library and a separate microservice (each with a distinct, ongoing cost profile) when extraction is actually warranted.
---

# Code Duplication Tradeoffs Instructions

When supporting architecture design, use this tool to help teams evaluate code
duplication across independently-owned codebases as a real tradeoff between
delivery speed and coordination cost — rather than reflexively applying the DRY
principle — and, when extraction is genuinely warranted, choose deliberately
between a shared library and a separate microservice based on structural
criteria rather than habit or fashion.

---

## Purpose

This tool helps the AI assistant by:

- treating DRY as a tradeoff to evaluate, not a rule to apply unconditionally,
  especially across codebases or teams that currently operate independently,
- using Amdahl's law as the reasoning tool for *why* removing duplication has a
  cost: the fraction of work that must be synchronized between teams caps the
  speedup available from adding people, and shared code across a team boundary
  is exactly that kind of synchronized work,
- recognizing when duplication is the right call — independent evolution speed
  outweighs the cost of occasional drift between copies,
- distinguishing a shared library from a separate microservice as two
  structurally different extraction paths, each with its own ongoing cost, not
  interchangeable implementations of "remove the duplication,"
- treating security- and correctness-critical logic (authentication, payment
  calculation, cryptography) as the sharpest exception to "duplication is
  fine," since the risk of the same mistake being made independently twice
  usually outweighs the coordination cost of sharing it.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- explicitly weighs synchronization/coordination cost against duplication cost
  before recommending removing duplication, rather than treating removal as
  self-evidently correct,
- when removal is warranted, chooses between a shared library and a separate
  microservice using stated structural criteria (dependency footprint, need
  for independent scaling/deployment, cross-language requirements, whether the
  logic is a real business domain) rather than defaulting to whichever
  approach is currently fashionable,
- flags correctness- or security-critical logic as a case where extraction is
  worth its coordination cost even when the general case would tolerate
  duplication,
- surfaces "who owns this now" explicitly for either extraction path, since
  both create a new artifact that someone must maintain on an ongoing basis.

---

## Instructions for the AI

1. **Frame duplication as a tradeoff, not a defect**
   - Push back on the reflex to extract shared code the moment two codebases
     look similar. Ask what coordination cost extraction would introduce
     between the teams currently duplicating the logic.
   - Apply Amdahl's law explicitly: the portion of work that must be
     synchronized between teams caps how much a team's throughput improves
     from adding people. A shared piece of code that both teams depend on is
     synchronized work — every change to it needs coordination across every
     team that consumes it.
   - Note the flip side: independently duplicated implementations are 100%
     parallel work with no synchronization tax, at the cost of possible drift
     and duplicated bug-fixing effort.

2. **Weigh the real cost of independent duplication**
   - Loss: a bug fixed in one copy doesn't propagate to the other; there's no
     automatic cross-team knowledge sharing when one team learns something the
     hard way.
   - Gain: neither team blocks on the other, and no cross-team review or
     coordination is required for changes to the duplicated logic.
   - Call out the sharp exception explicitly: for correctness- or
     security-critical logic (auth, payment calculation, cryptography), the
     risk of an independently-introduced mistake in either copy usually
     outweighs the coordination cost — recommend extraction here even when the
     general tolerance-for-duplication argument would otherwise apply.

3. **When extraction is warranted, choose deliberately between a library and a microservice**
   - **Shared library:** cheaper to build and deploy (package + repository
     manager), but couples every consumer at the dependency-version level —
     transitive dependency conflicts are a real, recurring cost. Requires the
     same language/runtime family across all consumers, and needs ongoing
     documentation and test investment, since a library's usage patterns
     aren't self-evident the way an API contract is.
   - **Separate microservice:** consumers integrate through an API only (a
     genuine black-box boundary, usable cross-language), but requires a full
     deployment/monitoring/on-call lifecycle, adds network latency and a new
     failure mode (partial availability, cascading failure) to every caller,
     and shifts resource consumption from the caller to the new service —
     relevant to that service's own capacity planning.
   - Decision heuristic to apply: does the shared logic form a genuine,
     independent business domain (its own data, its own lifecycle, a natural
     owner)? Favor a **microservice**. Is it small, largely stateless,
     dependency-light logic that runs inline with the caller's own request? Favor
     a **library**. If neither owner exists yet but the correctness risk from
     section 2 applies, extract anyway and treat naming an owner as part of
     the recommendation, not an afterthought.

4. **Account for the ongoing cost of either path, not just the extraction cost**
   - Both paths create a new artifact someone must own indefinitely:
     deployment process, documentation, tests, version-compatibility
     guarantees.
   - The first shared library or service in an organization is expensive to
     set up (tooling, process, conventions); each subsequent one is
     progressively cheaper. Factor this into an "extract now vs. wait"
     recommendation — a first extraction pays a fixed setup cost that later
     ones won't.
   - For a microservice specifically, prompt for the latency/SLA impact
     explicitly (does adding a network hop risk this caller's SLA?) and ask
     for a concrete mitigation plan — retry with backoff, a circuit breaker,
     caching — rather than treating the added network hop as free.

---

## AI decision guidance

When generating duplication guidance, keep these principles in mind:

- **Duplication across independently-owned codebases is a legitimate default**
  when the coordination cost of removing it exceeds the cost of occasional
  drift — not a violation to be corrected on sight.
- **"Library vs. microservice" is a structural decision, not a style
  preference** — driven by cross-language needs, independent
  scaling/deployment requirements, and whether the shared logic is a genuine
  business domain.
- **Correctness- and security-critical logic is the sharpest exception** to
  tolerating duplication — recommend extraction there even when coordination
  cost is real and significant.
- **Every extraction creates a new owned artifact with an ongoing maintenance
  cost** — never present extraction as a one-time, free improvement.

---

## Success criteria

A strong response should ensure that it:

- **identifies the actual coordination cost** the duplication in question is
  avoiding, not just the abstract DRY principle,
- **distinguishes library vs. microservice** using the structural criteria
  above, not habit or trend,
- **flags correctness-critical logic** as a special case warranting extraction
  despite coordination cost,
- **surfaces ownership and ongoing maintenance cost** for whichever extraction
  path is ultimately chosen.

---

## Example prompts for the AI

- "Two of our teams have near-identical auth logic in their services — should
  we extract it?"
- "We're deciding between extracting this shared logic as a library or a
  microservice — how do we choose?"
- "Our teams keep duplicating a currency-conversion helper — is that actually
  a problem worth fixing?"

---

## Related guidance

Use this tool alongside:

- architecture-parallelism
- architecture-decomposition
- architecture-composition
- architecture-inheritance-coupling-tradeoffs
- architecture-exception-design-and-anti-patterns
- architecture-flexibility-complexity-tradeoffs
- architecture-configuration-surface-tradeoffs
- architecture-third-party-dependency-conflicts — the same transitive-dependency coupling cost, from the perspective of consuming a library rather than deciding whether to extract one
