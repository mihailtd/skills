---
name: architecture-stages-of-change
description: Instructs the AI assistant to guide a proposed change through its motivational, conceptual, and detailed stages — allowing legitimate backtracking to an earlier stage as understanding deepens — and to scale process rigor to the significance of the change rather than to whether it's formally "architectural."
---

# Stages of Change Instructions

When supporting architecture design, use this tool to help teams work a
proposed change through motivation, concept, and detail in order, recognize
when backtracking to an earlier stage is the right move (not a failure), and
calibrate how much process rigor a change actually deserves.

---

## Purpose

This tool helps the AI assistant by:

- structuring change discussions around three stages — motivational,
  conceptual, detailed — so the underlying "why" is established before
  debating a specific approach, and the approach before the implementation
  details,
- normalizing backtracking to an earlier stage when new understanding
  surfaces, instead of treating it as wasted work,
- warning against the sunk-cost fallacy when a change already has
  significant conceptual or detailed-stage investment behind it,
- scaling the rigor applied to a change to its actual scope/significance,
  rather than to a formal (and often unknowable in advance) label of
  "architectural" versus "design" change.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- can articulate, for any change under discussion, which stage it's
  currently at (motivational, conceptual, or detailed) and what's still
  missing to progress,
- catches changes that jumped straight to "what we're going to build"
  without an established motivation, and pulls the discussion back to
  justify the "why" first,
- treats a change that reveals new information at the detailed stage as
  a signal to revisit the conceptual approach, not as a problem to push
  through anyway,
- applies review rigor proportional to a change's actual scope and impact,
  not proportional to a debate over whether it's "really" an architecture
  change,
- avoids continuing down a stage-appropriate path purely because of prior
  investment in it (the sunk-cost fallacy).

---

## Instructions for the AI

1. **Identify and separate the three stages explicitly**
   - **Motivational** — why do we want to make this change? Press for a
     specific problem, need, or opportunity being addressed, not a vague
     "it would be nice."
   - **Conceptual** — what do we think we will change? This is the level
     of approach: replacing one technology, optimizing a code path,
     adopting a different algorithm. Don't let this stage collapse into
     implementation detail prematurely.
   - **Detailed** — how are we going to make this change, including the
     transition from old state to new? This is where the real
     implementation complexity often lives (e.g., data migration), and it
     is frequently underestimated relative to the conceptual stage.

2. **Expect — and permit — non-linear progress**
   - Recognize that discussions often start at the conceptual stage (a
     proposed solution) with the motivation left implicit; when this
     happens, explicitly ask for the motivation before evaluating the
     concept, since a concept can't be properly evaluated without knowing
     what it's meant to achieve.
   - Recognize that detailed-stage work often surfaces new understanding
     that should send the discussion back to reevaluate the conceptual
     approach — treat this as the process working correctly, not as
     failure or wasted effort.
   - Explicitly name the sunk-cost fallacy when a team seems reluctant to
     revisit an earlier stage purely because of the effort already spent
     downstream — the cost already spent is not a reason to proceed with
     a now-questionable approach.

3. **Scale rigor to scope, not to a type label**
   - Avoid spending time trying to definitively classify a change as
     "architectural" versus "design" before deciding how much process to
     apply — this distinction is frequently unknowable until well into the
     work.
   - Instead, assess the change's actual scope and potential impact (how
     many components, how many relationships, how much of the system's
     current organization it touches or violates) and scale process rigor
     to that.
   - For changes that turn out to require the system's fundamental
     organization to evolve, apply the full weight of architectural
     process. For changes realizable within the current architecture,
     scale the process down accordingly — the practices are the same in
     kind, just different in degree.

---

## AI decision guidance

When generating change-process guidance, keep these principles in mind:

- **Motivation before concept, concept before detail:** don't let a
  discussion evaluate a solution before its problem is established, or
  finalize an approach before its transition mechanics are understood.
- **Backtracking is progress, not failure:** deeper understanding
  discovered downstream is valuable information — use it, don't suppress
  it to protect prior decisions.
- **Watch for the sunk-cost fallacy specifically:** call it out by name
  when it appears to be driving a decision to continue rather than the
  merits of the current approach.
- **Scope drives rigor, not classification:** don't let a debate over
  whether something counts as "architecture" substitute for actually
  assessing its impact.

---

## Success criteria

A strong stages-of-change response should help teams achieve:

- **explicit motivation:** every change proposal traceable to a specific,
  articulated reason for wanting it,
- **evaluated concepts:** approaches assessed against their actual
  motivation, not accepted at face value,
- **surfaced transition complexity:** the detailed "how do we get from old
  to new" work done deliberately, not glossed over,
- **healthy backtracking:** a team culture where revisiting an earlier
  stage is normal and encouraged when warranted, not stigmatized,
- **proportional process:** rigor applied based on a change's actual scope
  and impact, not on an early, often-premature architecture/design label.

---

## Example prompts for the AI

- "We jumped straight to proposing a solution for this — help me back up
  and articulate what problem we're actually trying to solve."
- "We've invested two weeks in this implementation approach and just
  found an issue — should we revisit the design, or push through?"
- "How much process rigor does this specific change actually warrant?"

---

## Related guidance

Use this tool alongside:

- architecture-capability-trajectory
- architecture-investment-mindset
- architecture-risk-management
- architecture-change-proposals
- architecture-alternative-generation
