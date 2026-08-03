---
name: architecture-parallelism
description: Instructs the AI assistant to design decompositions that support parallel team execution — favoring separability at upper layers of the system (services, applications) where isolated work outweighs coordination overhead, and using observed team coordination patterns as a live diagnostic for decomposition quality (heavy cross-team coordination signals a bad boundary).
---

# Parallelism Instructions

When supporting architecture design, use this tool to help teams design
decompositions that let people, not just problems, be divided
effectively — favoring separability where parallel work genuinely pays off,
and treating the coordination patterns that emerge once teams are assigned
to a decomposition as real, actionable feedback on whether the design's
boundaries are actually clean.

---

## Purpose

This tool helps the AI assistant by:

- distinguishing incrementalism (organizing work over time) from
  parallelism (organizing work over people), and treating decomposition
  quality as the shared enabler of both,
- explaining that parallelism pays off when a decomposition's subproblems
  are genuinely separable with clean interfaces — the more separable, the
  more efficiently a team can divide the work,
- calibrating where parallelism is actually worth pursuing: it's generally
  accessible and valuable at upper layers of a system's decomposition
  (applications, services) where isolated work clearly outweighs
  coordination overhead, and generally not worth pursuing at lower layers
  (classes, methods) where that ratio flips,
- using observed team coordination patterns as a live diagnostic for
  decomposition quality — low cross-team coordination between two areas is
  a positive signal about their interface; frequent, heavy cross-team
  coordination is a clear signal the decomposition boundary between them
  is drawn wrong,
- recommending that teams actually revisit and adjust a design when its
  assigned teams reveal, through their real coordination behavior, that a
  boundary isn't working — even if the written design looked fine on paper.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- explicitly separates "how do we sequence this work over time"
  (incrementalism) from "how do we divide this work across people"
  (parallelism), while recognizing both depend on the same underlying
  decomposition quality,
- assigns parallel work along decomposition boundaries that are genuinely
  separable, with clean, low-overhead interfaces between the pieces
  assigned to different people or teams,
- reserves parallelism for layers of the decomposition where it clearly
  pays off (e.g., splitting a web application team from a backend services
  team) and avoids forcing it at layers where coordination overhead would
  dominate any benefit (e.g., parallelizing within a single class),
- treats the amount of coordination that naturally emerges between teams
  assigned to different parts of a decomposition as real evidence about
  that decomposition's quality, not just an organizational side effect,
- is willing to recommend revisiting a design when the coordination
  patterns between teams reveal a boundary problem the written design
  didn't make apparent.

---

## Instructions for the AI

1. **Distinguish parallelism from incrementalism, but connect both to decomposition**
   - Frame incrementalism as organizing work over time (see
     architecture-incremental-design) and parallelism as organizing work
     over people — different axes, but both fundamentally enabled by the
     same thing: a decomposition with genuinely separable, cleanly
     interfaced pieces.

2. **Assign parallel work along genuinely separable boundaries**
   - When recommending how to split work across a team, favor decomposition
     boundaries that are well defined and cleanly interfaced — the more
     separable the pieces and the cleaner the interfaces between them, the
     more efficiently parallel work will proceed.
   - Avoid recommending parallel assignment across a boundary that isn't
     genuinely separable — this creates coordination overhead that
     undermines the point of parallelizing in the first place.

3. **Calibrate where parallelism is actually worth pursuing**
   - Recognize that parallelism is generally more accessible and valuable
     at the upper layers of a system's decomposition — e.g., splitting a
     web application team from a backend services team along a clear
     component boundary, especially when the two areas also call for
     different skills or technical knowledge that naturally organizes teams
     around them.
   - Recognize that as decomposition moves down the stack — toward classes
     and methods — the ratio of isolated-work-value to
     coordination-overhead shrinks, and at some point (typically around the
     level of an individual class) parallelism stops being worthwhile.
     Don't recommend forcing parallel assignment at this level.
   - Frame the underlying test as: does the work that can be done in
     isolation outweigh the overhead of coordinating the interfaces and
     connections between the pieces? Apply this test explicitly rather than
     assuming parallelism is always beneficial simply because more people
     are available.

4. **Use team coordination patterns as a decomposition-quality diagnostic**
   - When teams are assigned to different parts of a decomposition,
     recommend actively observing how much coordination naturally occurs
     between them, and treat that as real, actionable signal about the
     decomposition itself.
   - Low, easy coordination between teams working on different parts (few
     meetings needed, a stable interface) is a positive signal — it
     indicates the decomposition created genuinely clean boundaries there.
   - Frequent, heavy coordination between two teams (constant meetings,
     an interface that keeps changing) is a clear signal that the
     decomposition boundary between their areas isn't actually clean, even
     if it looked reasonable on paper.
   - When this signal appears, recommend revisiting the design at that
     boundary specifically — the team behavior has surfaced a design
     problem that wasn't visible from the written decomposition alone.

---

## AI decision guidance

When generating parallelism guidance, keep these principles in mind:

- **Parallelism rides on decomposition quality, not on headcount:** more
  people working on a poorly separated decomposition doesn't produce
  effective parallelism — it produces coordination overhead.
- **Test isolation-value against coordination-cost explicitly:** don't
  assume parallel assignment pays off — check the actual ratio for the
  layer of decomposition in question.
- **Upper layers, not lower layers, are the natural home for team
  parallelism:** default parallel team assignment to service/application
  boundaries, not to classes or methods.
- **Team coordination is real design feedback:** treat unusually heavy
  cross-team coordination as a decomposition defect to fix, not just a
  process or communication issue to manage around.

---

## Success criteria

A strong parallelism response should help teams achieve:

- **clean parallel boundaries:** work divided across genuinely separable
  decomposition boundaries with low-overhead interfaces,
- **appropriately scoped parallelism:** parallel team assignment applied at
  layers where it clearly pays off, and avoided at layers where it would
  create net overhead,
- **diagnostic use of coordination data:** team coordination patterns
  actively monitored and used as evidence about decomposition quality,
- **design responsiveness:** decomposition boundaries revisited and
  adjusted when real team coordination behavior reveals a problem the
  written design didn't surface.

---

## Example prompts for the AI

- "We're splitting this system across two teams — help me figure out
  whether the decomposition boundary we've chosen will actually support
  that split well."
- "Two of our teams are in constant sync meetings about their shared
  interface — is that telling us something about our design?"
- "At what point does it stop making sense to split work across people for
  this part of the system?"

---

## Related guidance

Use this tool alongside:

- architecture-decomposition
- architecture-composition
- architecture-incremental-design
- architecture-code-duplication-tradeoffs
