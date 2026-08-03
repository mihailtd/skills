---
name: architecture-incremental-design
description: Instructs the AI assistant to work a decomposed design incrementally — solving and recomposing a subset of the decomposition tree into a working whole before returning to expand it further — to sustain motivation, make progress when the endpoint is unclear, discover when later increments turn out unnecessary, and resolve scope debates with delivered results instead of abstract argument.
---

# Incremental Design Instructions

When supporting architecture design, use this tool to help teams work
through a decomposed design incrementally rather than exhaustively —
solving and recomposing a subset of the design tree into working form,
using the results to inform what to tackle next, and letting delivered
increments settle scope debates that abstract argument can't.

Note: this is a design-execution technique — how to work through an already
decomposed problem over time. For managing the scope of change proposals
and architectural investment more broadly, see
architecture-incremental-delivery.

---

## Purpose

This tool helps the AI assistant by:

- framing the design process as traversal of a decomposition tree — each
  node a problem that can be broken into child problems, down to leaf
  nodes solved directly — and pointing out that a full, linear traversal
  before anything works is rarely the right approach for real product
  development,
- recommending working incrementally instead: solve and recompose a subset
  of the tree into a working whole, then return to expand it further in
  later increments,
- explaining the two distinct reasons incrementalism pays off — sustaining
  motivation and momentum through visible intermediate results, and making
  progress even when the full endpoint isn't yet known or fully understood,
- flagging that later increments sometimes turn out to be unnecessary once
  earlier ones are running — treating this as a valuable discovery, not a
  sign the plan was wrong from the start,
- offering incremental delivery as a practical way to resolve scope
  disagreements: instead of debating the right scope in the abstract, agree
  on an incremental plan and let delivered results inform the debate.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- treats the decomposition tree as something to work through incrementally
  by default, not as a checklist that must be fully completed before
  anything is usable,
- selects an initial subset of the design to solve and recompose into a
  working form, explicitly deferring the rest rather than trying to solve
  everything at once,
- uses early increments specifically to sustain team motivation and to
  generate real feedback when the eventual endpoint isn't fully clear,
- revisits later, still-undecomposed parts of the tree with the benefit of
  what was learned from earlier increments — including sometimes deciding
  those parts aren't needed at all,
- converts abstract scope disagreements into an incremental plan with
  checkpoints, so the disagreement can be resolved with delivered evidence
  instead of continued argument.

---

## Instructions for the AI

1. **Frame design as tree traversal, but reject full linear traversal by default**
   - Describe the design process as traversing a decomposition tree: each
     node is a problem decomposable into child problems, continuing down to
     leaf nodes that are solved directly (see architecture-decomposition).
   - Point out that fully traversing the tree — solving every node before
     anything works — might be reasonable for a small, solo effort, but is
     rarely the right default for real product development, which wants
     both speed and visible intermediate results.

2. **Work a subset of the tree into a working whole, then return to expand it**
   - Recommend explicitly choosing a subset of the decomposition to solve
     and recompose into working form now, while deliberately deferring the
     rest of the tree to later increments.
   - Make clear that later increments can return to any deferred node and
     expand it further — the tree doesn't need to be fully resolved in a
     single pass, and doesn't need to be resolved in any fixed order.

3. **Use incrementalism to sustain motivation**
   - When a design effort risks losing momentum from a long stretch without
     anything working, recommend restructuring the work incrementally
     specifically to produce visible, working intermediate results sooner —
     frame this as a legitimate driver of technical sequencing decisions,
     not just a nice-to-have.

4. **Use incrementalism when the endpoint is unclear or unknown**
   - When the team doesn't yet know all the problems to solve, or knows the
     problems but can't yet solve all of them, recommend separating out the
     well-understood parts, getting those working, and using the feedback
     from that result to inform what's tackled next.
   - Treat this as the primary technique for making real progress under
     genuine uncertainty about scope or approach, rather than stalling on
     upfront planning that the team doesn't yet have enough information to
     do well.

5. **Treat "this later increment wasn't needed after all" as a valid, valuable outcome**
   - When earlier increments are running, explicitly check whether the
     later, still-planned increments are still needed — sometimes an
     element that seemed necessary in the abstract turns out to be
     unnecessary once the working system reveals it wasn't actually
     required.
   - Frame this as the incremental approach paying off by avoiding wasted
     effort, not as evidence the original design was flawed.

6. **Use incremental delivery to resolve scope disagreements**
   - When team members disagree about scope (e.g., a full implementation
     versus a minimal one), recommend not debating the abstract question
     directly — instead, get agreement on an incremental plan and check in
     after each increment is delivered.
   - Point out that concrete, delivered results tend to make it easier for
     both sides to find common ground than continued argument about a
     hypothetical scope.

---

## AI decision guidance

When generating incremental-design guidance, keep these principles in mind:

- **Full traversal is the exception, not the default:** always ask whether
  the design can be worked incrementally before assuming the whole
  decomposition tree needs to be resolved up front.
- **Pick a real, working subset, not a partial mess:** an increment should
  recompose into something functional, not just a fragment of unfinished
  pieces.
- **Match the reason to the situation:** distinguish "we need visible
  progress to stay motivated" from "we don't yet know the full scope" —
  both justify incrementalism, but call for slightly different sequencing
  choices.
- **Later increments are provisional, not guaranteed:** revisit whether a
  deferred increment is still needed once earlier increments provide real
  information, rather than treating the original plan as fixed.
- **Deliver to resolve disagreement, don't just argue longer:** when scope
  is contested, default to proposing an incremental plan with checkpoints
  over continuing an abstract debate.

---

## Success criteria

A strong incremental-design response should help teams achieve:

- **working increments, not partial fragments:** each increment recomposes
  into something functional, even if it doesn't cover the whole design,
- **sustained momentum:** visible progress maintained through intermediate,
  working results rather than a long stretch of nothing working,
- **progress under uncertainty:** forward movement even when the full
  eventual scope isn't yet known, driven by feedback from completed
  increments,
- **avoided waste:** later increments re-evaluated (and sometimes dropped)
  based on what earlier increments revealed,
- **resolved scope debates:** disagreements about scope settled by
  delivered, concrete results rather than prolonged abstract argument.

---

## Example prompts for the AI

- "We've decomposed this design into a dozen pieces — help me figure out
  which subset to build first so we have something working soon."
- "We don't fully know the scope of this feature yet — how should we
  sequence the work so we can make progress anyway?"
- "The team is arguing about whether we need the full version of this or a
  minimal one — help me turn that into an incremental plan instead of more
  debate."

---

## Related guidance

Use this tool alongside:

- architecture-decomposition
- architecture-composition
- architecture-parallelism
- architecture-incremental-delivery
