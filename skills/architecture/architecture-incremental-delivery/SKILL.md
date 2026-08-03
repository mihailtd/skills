---
name: architecture-incremental-delivery
description: Instructs the AI assistant to keep architectural change incremental by default — recognizing and bounding "while we're here" scope creep, naming the two failure modes (open-ended re-architecture vs. short-term retrenchment), and organizing change into a long-term vision, a backlog of potential changes, and a smaller set of current work that always ships.
---

# Incremental Delivery Instructions

When supporting architecture design, use this tool to keep large,
open-ended "re-architecture" efforts from displacing steady incremental
progress — catching scope creep as it happens, naming the two ways teams
typically fail to manage it, and organizing change into vision, backlog,
and current work so that something always ships.

---

## Purpose

This tool helps the AI assistant by:

- treating incremental, tactical change as the default mode of delivery,
  reserving large-scale re-architecture for the rare cases that genuinely
  require it,
- explaining why big changes are systematically mis-assessed — their costs
  get underestimated and their benefits overestimated as scope grows — and
  why that means a big change needs an outsized, not just adequate, return
  to be a good investment,
- recognizing the specific "while we're here" scope-creep dialogue pattern
  (a small change ripples from component to component) as it happens, and
  treating it as useful ideation to capture, not a plan to execute wholesale,
- naming the two common failure modes when scope balloons — going all-in on
  an ever-expanding change that never ships, versus retrenching to
  short-term fixes that undermine the larger change — so teams can
  recognize and avoid both,
- providing a concrete organizing structure (long-term vision, backlog of
  potential changes, current work) that lets ambitious change happen without
  requiring it all at once.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- defaults to proposing the narrowest change that makes real progress,
  reserving broad re-architecture proposals for cases with a clearly
  articulated, outsized return,
- catches "since we're in here, we might as well also fix..." moments
  during design discussions, and explicitly separates them from the change
  actually being delivered,
- treats scope-expansion discussions as valuable for generating options and
  surfacing documentation/understanding gaps (e.g., "did we know changing B
  would affect C and D?"), without treating every generated option as
  something to build now,
- avoids both failure modes: an ever-growing change that ships nothing, and
  a short-term fix that quietly makes the eventual real change harder,
- maintains three distinct, legible categories at all times — a long-term
  vision, a backlog of potential changes, and the current, smaller set of
  work actually being executed — so ambition and delivery don't have to
  compete with each other.

---

## Instructions for the AI

1. **Default to narrow, tactical, fast-executing changes**
   - Treat incremental delivery as the normal mode of operation; large
     re-architecture efforts should be infrequent, and each one needs to
     justify itself with an outsized return, not just a plausible one.
   - Explicitly flag the estimation trap: as proposed scope grows, teams
     systematically underestimate the true cost and overestimate the
     benefit — call this out directly when a change's scope is expanding
     mid-discussion, since that's exactly when the trap is most active.

2. **Recognize the "while we're here" scope-creep pattern**
   - Watch for the characteristic snowball: a change to component A touches
     its interface with B, which someone's "been wanting to revisit," which
     then ripples into B's relationships with C and D.
   - Don't shut this down reflexively — this kind of exploration has real
     value: it tests whether the team's understanding of the system's
     dependencies is accurate (if you don't know changing B affects C and D
     until you're halfway through, that's a costly discovery), and it
     surfaces genuinely useful new ideas, some good and some not.
   - But make clear that generating these ideas is different from committing
     to build them now — the next step is to sort them (see the
     backlog/current-work structure below), not to fold them all into the
     current change by default.

3. **Name both failure modes explicitly when scope is expanding**
   - **Going all-in:** the team keeps expanding scope because the new work
     is exciting and "everything will work out," while nothing gets
     delivered and no complete change ships. Call this out as a hallmark of
     immature project management, and redirect toward shipping something
     concrete.
   - **Retrenching to short-term fixes:** on recognizing the scope has grown
     too large, the team bypasses the real problem with a fix that costs
     the product net value and can make the eventual larger change *harder*
     to achieve later. Treat this as a poor investment in its own right —
     see architecture-investment-mindset for why "short-term" doesn't mean
     "low-cost."
   - When either pattern shows up, name it directly and propose the
     three-category structure below as the way out of the see-saw between
     these two extremes.

4. **Organize change into three explicit categories**
   - **Long-term vision:** the desired fundamental organization of the
     system and the argument for it, contrasted with the current state.
     This should stay directional, not detailed — it exists to guide
     decisions, not to be a spec.
   - **Backlog of potential changes:** the set of changes that would move
     the system from its current state toward the vision. These are
     *potential* — capturing ideation without committing to execution —
     and this is exactly where scope-creep ideas from step 2 should land
     when they're good but not needed right now.
   - **Current work:** the smaller, concrete set of changes actually being
     executed right now. Every item here should be moving the system toward
     the vision (otherwise it's probably a bad investment — see
     architecture-investment-mindset), but current work is never expected
     to cover the entire vision in one pass.
   - When current work threatens to grow mid-flight (the "shouldn't we just
     re-architect this while we're here" moment), use the vision as the
     shared reference point to debate the proposed expansion, then sort the
     outcome explicitly: most of it to the backlog for later, a smaller
     piece — if genuinely warranted — into current work.

5. **Treat the bookkeeping overhead as worth it**
   - Acknowledge that maintaining three categories takes some discipline,
     but make the payoff explicit: it removes the pressure to cram every
     good idea into the current change (there's a legitimate place for it —
     the backlog), while also preventing the long-term vision from being
     discarded just because current work is managed separately.
   - Recommend keeping system documentation current as part of this
     practice — knowing in advance that changing B will affect C and D is
     what makes scope conversations productive instead of a source of
     mid-project surprises.

---

## AI decision guidance

When generating incremental-delivery guidance, keep these principles in
mind:

- **Narrow and shipping beats broad and stalled:** default to the smallest
  change that makes real progress toward the vision.
- **Bigger scope, bigger scrutiny:** treat scope growth mid-discussion as a
  trigger to re-examine cost and benefit estimates specifically, since both
  become less reliable as scope grows.
- **Capture ideas, don't automatically commit to them:** scope-creep
  discussions are valuable for ideation and for testing system
  understanding — route the output to the backlog rather than either
  discarding it or building all of it now.
- **Name the failure mode you're seeing:** when a discussion is drifting
  into open-ended re-architecture or into a short-term fix, say so directly
  and reground the conversation in vision / backlog / current work.
- **Vision guides, it doesn't dictate:** use the long-term vision as the
  shared reference for evaluating whether a proposed scope expansion is
  worth pulling into current work, not as a checklist to complete
  immediately.

---

## Success criteria

A strong incremental-delivery response should help teams achieve:

- **shipping cadence:** a steady stream of narrow, completed changes rather
  than a single large effort perpetually in progress,
- **bounded scope discussions:** "while we're here" ideation captured and
  sorted, not either suppressed or executed wholesale,
- **named failure modes:** explicit recognition when a discussion is
  trending toward open-ended re-architecture or toward a costly short-term
  fix, with a concrete redirect,
- **legible categories:** a clearly maintained distinction between
  long-term vision, backlog, and current work that the team can point to at
  any time,
- **documentation that pays off:** system understanding accurate enough
  that dependency ripple effects (A affects B affects C) are known in
  advance rather than discovered mid-project.

---

## Example prompts for the AI

- "This 'quick fix' to component A is starting to pull in changes to B, C,
  and D — help me figure out what to actually do now versus defer."
- "We keep bouncing between doing the full re-architecture and doing a
  band-aid fix — how do we find a middle path we can actually ship?"
- "Help me set up a long-term vision, backlog, and current-work structure
  for this system so ambitious ideas have somewhere to go without stalling
  delivery."

---

## Related guidance

Use this tool alongside:

- architecture-investment-mindset
- architecture-evolution-cadence
- architecture-stages-of-change
- architecture-capability-trajectory
- architecture-change-proposals
- architecture-alternative-generation
- project-management-adaptive-project-management
