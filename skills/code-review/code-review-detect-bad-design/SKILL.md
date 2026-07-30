---
name: code-review-detect-bad-design
description: Guides AI reviewers to detect functional design smells and architectural debt in code structure.
---

# Code Review: Detect Bad Design

This skill helps AI reviewers identify functional design smells and structural
pathologies that indicate architectural debt or brittle code. It focuses on
module interactions, dependency quality, component cohesion, and interface
clarity.

---

## When to use this skill

Use this skill when you need to:

- find functional design smells in code or architecture,
- evaluate whether a functional system has structural rot or hidden coupling,
- call out risky patterns like cyclic dependencies, mashed components, and bossy modules,
- recommend practical refactorings to restore healthy module boundaries,
- preserve one-way data flow and clean composition in a functional-lite design.

---

## Outcome

Produce a review that:

- identifies the specific design smells present,
- explains why each smell matters for maintainability,
- recommends concrete refactoring strategies,
- validates that the code preserves clear data flow and modularity,
- warns against architectural debt before it becomes pervasive.

---

## Instructions for the AI

1. **Treat design smells as symptoms**
   - Identify the underlying structural pathology, not just the surface code issue.
   - Use the smell taxonomy to prioritize the deepest architectural problems.

2. **Detect cyclic dependencies**
   - Model module interactions as a graph and look for closed loops.
   - If a cycle exists, recommend breaking it by merging cohesive logic or using
     higher-order functions / dependency injection.

3. **Detect feature density**
   - Flag modules or functions that do too much or combine unrelated responsibilities.
   - Recommend decomposing dense logic into smaller pure functions and using
     pipe-and-filter composition.

4. **Detect unstable dependencies**
   - Identify stable code that depends on volatile or experimental modules.
   - Recommend replacing the dependency or isolating it behind an adapter.

5. **Detect mashed components**
   - Look for a single feature spread across several modules,
     requiring shotgun surgery to change.
   - Recommend regrouping the feature into a cohesive module boundary.

6. **Detect ambiguous interfaces**
   - Flag vague function signatures, generic parameter shapes, or overloaded entry points.
   - Recommend specific input/output contracts, ADTs, or clearer type definitions.

7. **Detect mesh components**
   - Look for dense dependency graphs with no clear hierarchy or coordinator.
   - Recommend introducing a facade or coordinator function to simplify interactions.

8. **Detect first lady components**
   - Flag huge modules that implement many unrelated behaviors themselves.
   - Recommend splitting the module into smaller specialized modules and delegating
     responsibilities to shared utilities.

9. **Detect bossy components**
   - Flag thin delegators that add no meaningful logic and only forward calls.
   - Recommend removing the unnecessary layer or collapsing it into the caller.

10. **Validate the overall functional health**
    - Check that the system preserves a one-way flow of data and avoids hidden side effects.
    - Look for explicit boundaries between pure functions and side-effecting code.

---

## Decision points and guidance

- **Can the code be represented as a DAG?** If not, the structure is suspect.
- **Is a component doing more than one responsibility?** If yes, recommend splitting.
- **Does stable code depend on an unstable module?** If yes, isolate or replace it.
- **Is a feature scattered across many files?** If yes, consolidate it.
- **Are public interfaces specific and meaningful?** If not, sharpen them.
- **Is the module graph a web or a tree?** If it's a mesh, add a coordinator.
- **Does the code contain a massive god module?** If yes, break it into specialist modules.
- **Does the architecture contain empty delegation layers?** If yes, remove them.

---

## Quality criteria

A strong bad-design review should confirm that the codebase is:

- **acyclic:** module dependencies flow in one direction,
- **cohesive:** each component has a single clear purpose,
- **stable:** core modules do not depend on fragile dependencies,
- **localized:** related logic lives together,
- **clear:** interfaces are precise and expressive,
- **coordinated:** interactions are managed through explicit composition,
- **lean:** no massive monoliths or empty delegation layers remain,
- **healthy:** the design supports evolution without shotgun surgery.

---

## Review checklist

Use these questions during the review:

- [ ] Are there any cyclic dependencies between modules?
- [ ] Does any function or module combine multiple unrelated responsibilities?
- [ ] Is there stable code depending on an unstable or experimental dependency?
- [ ] Is a single feature scattered across many different modules?
- [ ] Are public interfaces vague, generic, or overloaded?
- [ ] Does the module interaction map resemble a web rather than a tree?
- [ ] Does any module act as a massive god module with too many responsibilities?
- [ ] Does any module exist only to delegate without adding value?
- [ ] Do pure functions remain separated from side-effecting code when appropriate?
- [ ] Is the overall architecture easy to explain and reason about?

---

## Example prompts

- "Review this code for functional design smells and tell me what structural debt you see."
- "Identify any cyclic dependencies or mesh-like coupling in this module graph."
- "Can you find places where a function is too dense or a module is too bossy?"

---

## Related guidance

This skill complements:

- code-review-code-structure
- code-review-data-structures
- architecture-building-with-failure-in-mind
- architecture-risk-management