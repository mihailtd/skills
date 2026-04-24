---
name: code-review-architecture
description: Guides AI reviewers to evaluate software architecture for functional integrity, structural health, and maintainability.
---

# Code Review: Architecture

This skill helps AI reviewers assess whether software architecture is healthy,
clear, and aligned with the system's requirements. It focuses on structural
modularity, data flow, component responsibility, and the architectural
behaviors that guard against long-term debt.

---

## When to use this skill

Use this skill when you need to:

- review architecture for structural soundness,
- detect functional integrity problems before they become brittle debt,
- validate that implementation follows architectural intent,
- ensure the design supports MVP delivery and iterative growth,
- identify unhealthy coupling, unclear interfaces, or over-engineered abstractions.

---

## Outcome

Produce an architectural review that:

- diagnoses structural health issues,
- explains problems in terms of design smells and architectural debt,
- recommends practical refactors to restore clarity,
- verifies that the architecture supports clear data flow and maintainable growth,
- confirms that design choices are driven by the problem, not by the preferred tool.

---

## Instructions for the AI

1. **Validate problem definition and scope**
   - Confirm the architecture solves a clearly defined problem.
   - Check whether requirements and success criteria are explicit.
   - Favor vertical-slice MVP thinking over bottom-up layer-by-layer building.

2. **Evaluate architectural health indicators**
   - Problem Definition: Does the architecture have a clear goal and scope?
   - Architecture Validation: Is the design checked against functional and
     non-functional requirements?
   - Technological Fit: Are tools chosen to solve the problem, not because they
     are preferred?
   - Knowledge Sovereignty: Does the team have the right domain knowledge to
     own the architecture?
   - Systematic Processes: Is there a repeatable, actionable development
     approach?

3. **Assess modularity and composition**
   - Prefer composition over inheritance where possible.
   - Look for abstractions that are meaningful rather than wrappers for hidden
     complexity.
   - Flag forwarding layers that merely delegate without adding value.
   - Check that components are cohesive and follow one responsibility.

4. **Detect structural pathologies**
   - Feature Density: Flag modules or classes doing multiple unrelated jobs.
   - Cyclic Dependencies: Model component dependencies and break any loops.
   - Mashed vs. Mesh Components: Ensure single features are not scattered and
     modules are not tangled into a dense dependency web.
   - Bossy Components: Remove useless delegation layers that add cognitive overhead.

5. **Verify interfaces and API clarity**
   - Ensure public interfaces are specific and expressive.
   - Check that APIs do not hide wildly different behaviors behind one entry point.
   - Confirm error handling is explicit, consistent, and not silently swallowed.

6. **Review data structure and logic alignment**
   - Ensure chosen data structures match the most frequent operations.
   - Validate that ordering, uniqueness, and relationship requirements are
     modeled naturally.
   - Reject solutions that force data into inappropriate structures.
   - Encourage the simplest structure that satisfies requirements.

7. **Enforce maintainable review discipline**
   - Check for TODOs or FIX-ME comments that masquerade as design decisions.
   - Recommend SMART feedback with clear impact and remediation guidance.
   - Prioritize defects by their structural impact.

---

## Decision points and guidance

- **Is the problem clearly defined?** If not, the review should ask for a sharper
  statement before judging architecture.
- **Does the implementation match the architecture?** If not, identify the drift.
- **Are components composed instead of inherited unnecessarily?** If not, suggest
  a composition-based alternative.
- **Does the module graph have cycles or mesh-like coupling?** If yes, recommend
  breaking the cycles and simplifying interactions.
- **Is the API surface explicit and predictable?** If not, recommend clearer
  entry points and contracts.
- **Is the architecture designed for the most frequent use cases?** If not,
  recommend realigning the design to the 80/20 workload.
- **Is the architecture unnecessarily complex?** If so, recommend simplifying
  the design with the KISS principle.

---

## Quality criteria

A strong architecture review should confirm that the system is:

- **disciplined:** goals and scope are clear,
- **validated:** design choices are justified against requirements,
- **modular:** components are cohesive and composable,
- **explicit:** control flow and contracts are easy to understand,
- **maintainable:** the design avoids hidden coupling and brittle abstractions,
- **scalable:** the architecture supports growth without forcing drastic rewrites,
- **simple:** complexity is only used where it is necessary,
- **actionable:** feedback includes clear remediation steps.

---

## Review checklist

Use these questions during the review:

- [ ] Is the architectural goal and scope explicit?
- [ ] Does the implementation reflect the intended architecture?
- [ ] Are vertical-slice MVP concerns addressed rather than only horizontal layers?
- [ ] Is composition preferred over inheritance for flexible designs?
- [ ] Are cyclic dependencies absent from the component graph?
- [ ] Has feature density been avoided in modules and classes?
- [ ] Are APIs specific and not overloaded with unrelated behavior?
- [ ] Are error flows explicit and consistently handled?
- [ ] Are data structures aligned with the most frequent operations?
- [ ] Are unnecessary delegation layers removed?
- [ ] Are design decisions driven by requirements, not by tool preference?

---

## Example prompts

- "Review this architecture for structural health and tell me where it is brittle."
- "Identify any design smells that would make this system hard to maintain."
- "Does the implementation match the intended architecture, or has it drifted?"

---

## Related guidance

This skill complements:

- code-review-code-structure
- code-review-data-structures
- code-review-detect-bad-design
- architecture-risk-management
- architecture-building-with-failure-in-mind