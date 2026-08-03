---
name: architecture-decomposition
description: Instructs the AI assistant to break a complex design problem down recursively into a manageable, appropriately-sized set of elements — enough to actually simplify the problem, not so many that managing their relationships becomes the new problem — with each element abstracting away enough detail to be solved independently.
---

# Decomposition Instructions

When supporting architecture design, use this tool to help teams apply
decomposition deliberately — recognizing it as the same basic move used
throughout architecture practice, and evaluating candidate decompositions
against concrete criteria (element count, level of abstraction) rather than
accepting the first breakdown that comes to mind.

---

## Purpose

This tool helps the AI assistant by:

- framing decomposition as the default first step for any nontrivial design
  problem, and as the same fundamental technique already used elsewhere in
  architecture practice (components/relationships/environment, phased
  change processes) rather than a new skill specific to design,
- distinguishing the easier case (decomposing an existing system, where the
  breakdown is already given) from the harder case (designing a new system
  or making significant architectural changes, where the decomposition
  itself must be proposed and evaluated),
- providing concrete criteria for judging a candidate decomposition: does
  it produce a manageable number of elements, and does each element
  abstract away enough detail to be solvable independently,
- warning against both failure directions — too few elements (the
  decomposition didn't actually simplify anything) and too many elements
  (the relationships between them become a new, unmanaged problem),
- being honest that there's no universal formula for the "right" number of
  elements — it depends on the specific problem, even though a small
  number (order of a handful) is often a reasonable starting target.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- treats decomposition as the explicit first move on any design problem too
  large to solve directly, applied recursively until reaching pieces that
  are directly solvable,
- recognizes when a decomposition needs to be actively proposed and
  evaluated (new system, significant architectural change) versus when it's
  already given (working within an existing system's established
  boundaries),
- evaluates any candidate decomposition against two concrete criteria:
  element count and abstraction depth, not just intuition about what "feels
  right,"
- catches decompositions that under-divide (too few elements, no real
  simplification gained) or over-divide (too many elements, relationship
  management becomes the dominant cost),
- checks that each proposed element genuinely abstracts away enough detail
  to be worked on independently — a decomposition whose "pieces" still
  require deep knowledge of each other's internals hasn't actually
  isolated the subproblems.

---

## Instructions for the AI

1. **Apply decomposition as the default first step**
   - For any design problem too large or complex to solve directly, propose
     decomposition as the starting move: divide the problem so each
     resulting piece is smaller and more tractable, and apply this
     recursively until reaching pieces that can be solved directly.
   - Frame this explicitly as the same technique already in use elsewhere
     — describing a system as components, relationships, and environment is
     decomposition; breaking a change process into phases is decomposition
     — so a design problem doesn't call for a new technique, just applying
     the familiar one deliberately.

2. **Distinguish given decompositions from ones that must be proposed**
   - When working within an existing system, recognize that its
     decomposition into components and relationships is typically already
     established — the design task is to work within or extend it.
   - When designing a new system, or making changes significant enough to
     affect the system's fundamental organization, recognize that no
     decomposition is handed over — it must be actively proposed, and
     multiple candidate decompositions may need to be generated and
     compared (see architecture-alternative-generation).

3. **Evaluate candidate decompositions against element count**
   - Check whether the number of resulting elements is too small: if a
     decomposition yields only one or two elements, it likely hasn't
     simplified the problem — those elements will just need to be broken
     down further before real progress can be made.
   - Check whether the number is too large: if a decomposition yields many
     elements, point out that managing the relationships between them
     becomes a new, potentially harder problem than the one being solved.
   - Don't apply a fixed universal target (there is none), but treat a
     modest number — enough to create real division without becoming
     overwhelming — as a reasonable default to aim for, adjusted to the
     specific problem's actual complexity.

4. **Evaluate candidate decompositions against abstraction depth**
   - Check that each element in a candidate decomposition abstracts away
     enough internal detail that it can be understood and worked on
     without deep knowledge of the other elements' internals — this is
     what makes the subproblems genuinely separable.
   - If an element's design still requires close attention to another
     element's internal details to reason about correctly, treat that as a
     sign the decomposition boundary is drawn in the wrong place, not just
     an implementation inconvenience to work around.

---

## AI decision guidance

When generating decomposition guidance, keep these principles in mind:

- **Decomposition first, for anything nontrivial:** default to proposing a
  breakdown before attempting to solve a large problem directly.
- **Know which case you're in:** working within an established
  decomposition is a different task than proposing one from scratch —
  don't treat them the same way.
- **Judge by count and depth, not vibes:** always check a candidate
  decomposition against both the element-count and the
  abstraction-depth criteria explicitly.
- **Both directions are failure modes:** too few elements wastes the
  exercise; too many creates a relationship-management problem — watch for
  both, not just one.
- **No universal number, but a real target:** avoid pretending there's a
  formula, while still pushing for a specific, reasoned element count
  rather than an arbitrary one.

---

## Success criteria

A strong decomposition response should help teams achieve:

- **deliberate application:** decomposition explicitly invoked as the first
  step on any sufficiently complex design problem,
- **correct case recognition:** clarity on whether a decomposition is
  already given or needs to be proposed and evaluated,
- **right-sized element count:** a candidate decomposition with enough
  elements to genuinely simplify the problem, but not so many that their
  relationships dominate the effort,
- **genuine abstraction:** elements that can be reasoned about and worked
  on independently, without requiring deep knowledge of each other's
  internals,
- **recursive follow-through:** each resulting element decomposed further
  as needed, down to directly solvable pieces.

---

## Example prompts for the AI

- "This design problem feels too big to tackle directly — help me break it
  down into a manageable set of pieces."
- "We've decomposed this into a dozen small services — is that too many,
  and how would I know?"
- "Help me check whether these two proposed components actually isolate
  their concerns from each other, or if they're still too entangled."

---

## Related guidance

Use this tool alongside:

- architecture-composition
- architecture-incremental-design
- architecture-parallelism
- architecture-code-duplication-tradeoffs
- architecture-simplicity
- architecture-flexibility-complexity-tradeoffs
- architecture-alternative-generation
