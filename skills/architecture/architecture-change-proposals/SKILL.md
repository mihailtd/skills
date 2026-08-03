---
name: architecture-change-proposals
description: Instructs the AI assistant to capture every proposed change as a change proposal — a lightweight, progressively elaborated container spanning motivation, concept, and detail — tracked in an architectural backlog distinct from the product backlog, and to close the loop by redocumenting the system once a change is implemented.
---

# Change Proposals Instructions

When supporting architecture design, use this tool to help teams capture
proposed changes as change proposals from their earliest, roughest form,
track them through an architectural backlog distinct from the product
backlog, and close the loop by updating system documentation once a change
lands.

---

## Purpose

This tool helps the AI assistant by:

- treating the change proposal as a lightweight container that can start
  with just a sentence of motivation or a rough concept, and gets elaborated
  progressively rather than needing to be fully specified upfront,
- recommending that change proposals establish motivation and conceptual
  approach before committing to a detailed design, and that they capture
  which components/relationships are affected without prematurely
  specifying how the change will be realized,
- correcting the assumption that changes are always additive — proposals
  can just as legitimately remove, modify, or otherwise adjust existing
  parts of the system,
- distinguishing the architectural backlog (change proposals, tracked
  through motivation/concept/detail) from the product backlog (features and
  capabilities) — related, but not one-to-one, and maintained separately,
- treating "redocument the system" as the mandatory final step of any
  approved change — the loop isn't closed, and the next change can't start
  from an accurate baseline, until documentation reflects the new state.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- opens a change proposal as soon as a change is worth naming, even with
  minimal content, rather than waiting until it's fully thought through,
- resists over-specifying a change proposal early — establishing motivation
  and concept before detail, and only naming affected components/
  relationships at the concept stage rather than jumping to implementation
  specifics,
- considers non-additive changes (removal, modification) as naturally as
  additive ones when scoping a proposal,
- maintains a visible architectural backlog that tracks each proposal's
  current stage, and keeps it clearly distinct from the product backlog
  even where the two relate,
- treats updating system documentation as a required, not optional, closing
  step for every implemented change — and captures any further changes
  that surface during that update as new proposals rather than silently
  expanding the one just finished.

---

## Instructions for the AI

1. **Start proposals small and elaborate progressively**
   - Recommend opening a change proposal with as little as a sentence of
     motivation or a rough concept — the proposal is a container to gather
     information into as it develops, not a document that must be complete
     to exist.
   - Recommend that early revisions focus on big-picture questions: does
     this address a real requirement, and how does it align with the
     system's vision? Detailed realization mechanics come later, once
     motivation and concept have real alignment behind them.
   - At the concept stage, recommend naming which components or
     relationships will be affected without committing to precisely how the
     change will be implemented — that's a detailed-design concern (see
     architecture-stages-of-change).

2. **Remember changes aren't only additive**
   - When scoping a proposal, explicitly consider removal and modification
     of existing features or components as legitimate proposal types, not
     just new additions — product development's natural bias toward new
     features can make removal/modification proposals easy to overlook.
   - A proposal to adopt or revise an architectural principle, or to update
     the vision itself, is also a legitimate change proposal — capture it
     the same way, even though it may not lead to a detailed design (see
     step 4).

3. **Recognize meta-changes**
   - When a team wants to change its own change process, recommend handling
     that through a change proposal too — the process for managing change
     should itself be subject to the same process, which reinforces that
     any team member can propose any change and expect a fair evaluation.

4. **Maintain the architectural backlog, distinct from the product backlog**
   - Recommend a visible architectural backlog listing current, past, and
     future change proposals along with each one's current stage
     (motivation, concept, or detail) — this makes it possible to see at a
     glance whether a proposal has aligned on motivation before its concept
     is debated, and aligned on concept before detail is developed.
   - Keep the architectural backlog explicitly distinct from the product
     backlog: the product backlog describes features and capabilities; the
     architectural backlog describes change proposals. The two relate but
     are not one-to-one — one product backlog item can require several
     competing change proposals, and many product backlog items require no
     architectural change proposal at all.
   - For early-stage proposals (motivation/concept), the backlog entry
     itself can serve as the documentation — it doesn't need a separate,
     more expansive document until the proposal reaches detailed design.

5. **Redocument the system as the mandatory closing step**
   - Once a change proposal is approved and implemented, treat updating the
     system's documentation to reflect the new state as a required part of
     completing the change, not an optional follow-up.
   - Scale the documentation effort to the change's actual scope — a change
     that ended up narrow in impact needs only a modest update; a change
     with broader impact (e.g., a revised architectural principle) may
     require updates across multiple documents, including standards and
     other design documentation that reference it.
   - When updating documentation surfaces further changes worth making,
     capture those as new change proposals added to the backlog — don't
     fold them into the scope of the change just being closed out.
   - Recommend communicating the update to interested parties once
     documentation is current, so the new baseline is actually usable by
     the rest of the team.

6. **Use the pull-request analogy where it helps explain the process**
   - When it aids understanding, frame the change proposal lifecycle as
     analogous to a pull request: a baseline description of the system,
     a proposed "diff" from that baseline, a review/feedback period, and a
     "merge" that updates the baseline (here, the documentation) once
     approved — while noting that merging a change proposal into system
     documentation is typically far more manual than merging code.

---

## AI decision guidance

When generating change-proposal-process guidance, keep these principles in
mind:

- **Roughness is fine at the start:** don't demand full elaboration before a
  proposal is worth capturing — capture it, then develop it.
- **Motivation and concept before detail, always:** resist letting a
  proposal jump to implementation specifics before its "why" and "what" are
  aligned.
- **Removal and modification are first-class change types:** don't let the
  additive-features bias narrow what counts as a legitimate proposal.
- **Two backlogs, not one:** never conflate the architectural backlog with
  the product backlog, even when advising on how they relate.
- **No change is done until documentation says so:** treat redocumentation
  as part of the definition of "implemented," not a follow-up task that can
  slip.

---

## Success criteria

A strong change-proposal-process response should help teams achieve:

- **low-friction proposal creation:** proposals opened early and elaborated
  progressively, not blocked on being fully formed,
- **staged discipline:** proposals visibly showing which stage
  (motivation/concept/detail) they're in, with alignment at each stage
  before the next begins,
- **broadened proposal scope:** removal/modification and vision/principle
  changes captured as readily as new-feature changes,
- **a real architectural backlog:** distinct from the product backlog, with
  an explicit (not implied) relationship between the two,
- **closed loops:** every implemented change followed by documentation
  updates scaled to its actual impact, with any newly surfaced work
  captured as fresh proposals.

---

## Example prompts for the AI

- "We have a rough idea for a change but haven't thought through the
  details — help me capture it as a change proposal without over-specifying
  it yet."
- "How should we structure our architectural backlog so it doesn't get
  confused with the product backlog?"
- "We just finished implementing a change — what documentation actually
  needs to be updated, and how do we handle new ideas that came up while
  updating it?"

---

## Related guidance

Use this tool alongside:

- architecture-stages-of-change
- architecture-alternative-generation
- architecture-incremental-delivery
- architecture-urgent-vs-important
- architecture-backlog-management
- architecture-change-templates
- architecture-review-process
