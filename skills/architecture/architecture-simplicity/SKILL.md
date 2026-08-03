---
name: architecture-simplicity
description: Instructs the AI assistant to actively pursue architectural simplicity — favoring few, powerful, general abstractions over many specialized ones, detecting "tenuous invariants" (assumptions that hold today but aren't structurally enforced) as a concrete complexity red flag, resisting speculative future-proofing, and treating simplification as an ongoing responsibility rather than a one-time state.
---

# Simplicity Instructions

When supporting architecture design, use this tool to help teams recognize
what genuine simplicity looks like, detect complexity as it accumulates
(especially through unenforced assumptions), resist the urge to
future-proof against unknowns, and treat driving toward simplicity as
continuous work rather than a property a system either has or doesn't.

---

## Purpose

This tool helps the AI assistant by:

- defining simplicity correctly — few, powerful, general-purpose
  abstractions and relationships, not simply "fewer features" or "less
  powerful" — so it isn't confused with weakness,
- explaining why complexity is self-reinforcing (it degrades quality and
  velocity, and each new increment of complexity makes the next one easier
  to justify and harder to avoid),
- teaching the "tenuous invariant" as a concrete, checkable heuristic for
  finding complexity-in-waiting: an assumption the system currently relies
  on that isn't structurally guaranteed to keep holding,
- warning against future-proofing as a complexity trap — added flexibility
  for hypothetical future needs usually costs more than it returns, and
  simplicity itself is often the better hedge against an unknown future,
- framing active simplification (finding latent patterns, generalizing
  one-offs, retiring unneeded parts) as an ongoing architectural
  responsibility, not just a constraint on adding new complexity.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- can articulate why a proposed design is simple or complex in terms of the
  number and generality of its abstractions/relationships, not just a
  subjective feeling,
- actively looks for tenuous invariants in both new designs and existing
  systems, and treats finding one as a concrete, actionable finding, not a
  vague code smell,
- pressure-tests "future-proofing" proposals by asking what specific,
  credible future need justifies the added complexity now, rather than
  accepting speculative generality by default,
- treats simplification work (spotting patterns across previously
  one-off solutions, generalizing them, removing what's no longer needed)
  as a recurring item on the architecture team's agenda, not a one-time
  cleanup,
- recognizes when complexity has become self-reinforcing (new work is
  routinely justified by "well, it's already complex here") and calls it
  out explicitly.

---

## Instructions for the AI

1. **Define simplicity correctly: simple is not weak**
   - Explain simplicity as a property of *how few, how general, and how
     powerful* a system's abstractions and relationships are — not as a
     synonym for "does less" or "is underpowered."
   - Contrast a design with a small number of powerful, general-purpose
     abstractions against one with many narrow, specialized
     components/relationships that each handle a slightly different case —
     the former is simpler *and* can be just as capable, often more so,
     because the general abstraction covers cases the specialized ones don't.
   - Push back when "simple" is used dismissively to mean "not sophisticated
     enough" — the goal is fewer, stronger abstractions, not lesser ones.

2. **Explain why complexity compounds**
   - Make explicit that complexity has compounding costs: it degrades
     quality (more surface area for defects, harder-to-verify interactions)
     and velocity (more context required to make any change safely).
   - Highlight the self-reinforcing dynamic: once part of a system is
     already complex, adding more complexity there is easier to justify
     ("it's already messy") and structurally easier to do (more surface to
     attach to) — so complexity tends to accumulate fastest exactly where
     it's least wanted.
   - Use this dynamic to argue for addressing complexity early and
     proactively, rather than accepting "we'll clean it up later" as a safe
     default.

3. **Detect tenuous invariants as a concrete complexity signal**
   - Define a tenuous invariant as an assumption the system currently
     depends on for correctness or safety that holds *today* but is not
     structurally enforced — nothing in the architecture prevents it from
     becoming false.
   - Use the canonical example pattern: a write-through cache that's safe
     only because there's currently exactly one writer — the system relies
     on "only one writer" holding, but nothing stops a second writer from
     being added later, at which point the cache silently becomes unsafe.
   - When reviewing a design or an existing system, actively look for
     assumptions of this shape ("this works because X currently only
     happens one way") and flag them explicitly as tenuous invariants, not
     just as generic risks.
   - For each tenuous invariant found, recommend one of: enforce it
     structurally (make violation impossible or immediately detectable),
     document it prominently as a hard constraint future changes must
     respect, or redesign to remove the dependency on it altogether.

4. **Resist future-proofing as a complexity trap**
   - Treat proposals to add flexibility "in case we need it later" with
     skepticism by default — ask what specific, credible future requirement
     justifies the complexity being added *now*, using the trajectory
     model (see architecture-capability-trajectory) rather than vague
     possibility.
   - Point out that future-proofing usually backfires: the guessed-at
     flexibility often doesn't match the future need that actually
     materializes, so the system ends up carrying both the complexity of
     the unused flexibility *and* the complexity of whatever gets bolted on
     to handle the real, different need when it arrives.
   - Recommend, as the default hedge against an uncertain future, staying
     simple now and investing in the ability to change cheaply later —
     rather than trying to pre-build the specific future capability.

5. **Treat simplification as ongoing, active work**
   - Frame the architecture team's responsibility as not just "don't add
     unnecessary complexity" but actively "look for opportunities to reduce
     existing complexity" — the two are different postures, and both are
     needed.
   - Recommend a recurring practice of scanning for latent patterns across
     what are currently several similar, one-off solutions, and
     generalizing them into a single, more powerful shared abstraction when
     a genuine pattern is found (not before — see future-proofing caution
     above, since generalizing prematurely is its own complexity trap).
   - Recommend actively retiring components, code paths, or abstractions
     that are no longer needed, rather than letting them accumulate as
     unused-but-present complexity.

---

## AI decision guidance

When generating simplicity guidance, keep these principles in mind:

- **Simplicity is measured in abstractions, not features:** judge designs by
  how few, general, and powerful their abstractions are — don't equate
  "does more things" with "more complex" or "simpler" with "less capable."
- **Tenuous invariants are the concrete finding to hunt for:** when
  reviewing any design, explicitly ask "what does this rely on that isn't
  enforced?" and name what's found.
- **Complexity compounds — treat that as urgency, not just observation:**
  use the self-reinforcing dynamic as a reason to address complexity now
  rather than defer it.
- **Future-proofing needs a credible, specific justification:** don't accept
  speculative flexibility as free insurance — weigh its real cost against a
  concrete, trajectory-backed need.
- **Simplification is a verb, not a state:** recommend it as recurring
  practice (finding patterns, generalizing, retiring) rather than a project
  that finishes.

---

## Success criteria

A strong simplicity response should help teams achieve:

- **correctly framed simplicity:** designs evaluated by number/generality of
  abstractions, with "simple" never conflated with "weak,"
- **named tenuous invariants:** unenforced assumptions in a design or system
  identified explicitly, each with a recommended resolution (enforce,
  document, or redesign),
- **complexity urgency:** early intervention on emerging complexity,
  justified by its compounding, self-reinforcing cost,
- **rejected speculative flexibility:** future-proofing proposals held to a
  concrete-need bar, with simplicity favored as the default hedge,
- **an active simplification cadence:** a visible, recurring practice of
  pattern-finding, generalization, and retirement of unneeded complexity.

---

## Example prompts for the AI

- "We built a cache that assumes only one service writes to it — is that a
  tenuous invariant, and if so, what should we do about it?"
- "The team wants to add a configuration layer in case requirements change
  later — help me evaluate whether that's future-proofing worth doing."
- "We keep finding three or four near-identical one-off solutions across the
  codebase — should we generalize them into one abstraction?"

---

## Related guidance

Use this tool alongside:

- architecture-capability-trajectory
- architecture-investment-mindset
- architecture-reduce-risk
- architecture-risk-management
- architecture-flexibility-complexity-tradeoffs — this skill's speculative-future-proofing principle applied specifically to API extensibility mechanisms (dependency abstraction, hooks, listeners)
- architecture-configuration-surface-tradeoffs — the same principle applied to a dependency's configuration surface (passthrough vs. ownership)
- python-polars-vs-spark (package `python-core`) — the same principle applied to distributed-processing infrastructure: don't pay for horizontal scale a workload doesn't currently need
- architecture-idempotency-and-at-least-once-delivery — the minimal, escalation-ready building block for retry-safety this principle argues for keeping to, instead of reaching for CQRS/event-sourcing/multi-node coordination before they're needed
- architecture-delivery-semantics — the same principle applied to event/queue pipeline architecture: don't reach for a broker, multiple topics, or elaborate delivery machinery before the problem it solves is an actual, current one
