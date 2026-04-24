---
name: code-review-code-structure
description: Guides AI reviewers to evaluate code structure, detect architectural code smells, and recommend cleaner functional-lite designs.
---

# Code Review: Code Structure

This skill helps AI evaluate whether code is structurally sound and maintainable
by focusing on modularity, coupling, readability, and functional-lite design
principles. It warns against common structural code smells and encourages
practical, simple improvements.

---

## When to use this skill

Use this skill when you need to:

- review code for architecture and structure quality,
- identify and remove "spaghetti code" or tightly coupled designs,
- recommend improvements for maintainability and testability,
- ensure implementation aligns with architectural intent,
- evaluate a functional-lite codebase for clarity and cohesion.

---

## Outcome

Produce a code review that:

- identifies structural problems and their root causes,
- distinguishes between acceptable duplication and unhealthy coupling,
- recommends modular, cohesive, and loosely coupled alternatives,
- calls out unclear interfaces, hidden dependencies, and unstable modules,
- validates that control flow is explicit and conditional logic is complete.

---

## Instructions for the AI

1. **Look for code smells first**
   - Detect feature density: large functions or modules doing too many unrelated
     things.
   - Detect mashed components: single logic scattered across many files or
     layers.
   - Detect cyclic dependencies: modules depending on each other in a loop.
   - Detect mesh components: dense webs of interdependence with no clear
     structure.
   - Detect bossy components: thin delegators that simply orchestrate a long
     list of other functions.
   - Detect global state and undefined flows: globals, hidden side effects,
     and conditionals missing explicit else/fallback behavior.
   - Detect ambiguous interfaces: unclear, overlapping, or overly complex APIs.
   - Detect unstable dependencies: stable code depending on fragile or poorly
     structured modules.

2. **Evaluate modularity and cohesion**
   - Prefer components with a single, well-defined purpose.
   - Favor low dependency counts and simple interactions between modules.
   - Recommend splitting large, mixed-responsibility functions into smaller
     cohesive pieces when it improves clarity.

3. **Apply KISS and flat structure**
   - Prefer explicit, straightforward design over clever or nested abstractions.
   - Avoid deeply nested control flow or overly generic helper chains.
   - If the implementation is hard to explain, flag the structure as risky.

4. **Balance reuse with coupling**
   - Call out duplicated logic only when it causes maintenance risk.
   - Avoid extracting abstractions that create unnecessary dependencies.
   - Recommend reuse when it meaningfully reduces cognitive load and improves
     consistency.

5. **Assess testability**
   - Prefer code that can be isolated and reasoned about independently.
   - Flag functions or modules that are hard to unit test because they
     combine multiple concerns or rely on global state.
   - Recommend breaking code into smaller, injectable pieces for easier
     testing.

6. **Avoid premature optimization**
   - Emphasize clear design before performance tuning.
   - Question optimizations that complicate the structure without evidence of
     need.
   - Recommend keeping the code simple unless there is a measured performance
     requirement.

7. **Verify implementation matches architecture**
   - Check whether the code follows the intended architecture or whether it
     has drifted into ad hoc, tangled implementation.
   - Confirm that APIs are consistent, expressive, and aligned with the overall
     design.
   - Ensure error handling is explicit and not silently swallowed.

---

## Decision points and guidance

- **Is this component doing too much?** If yes, recommend splitting by logical
  responsibility.
- **Is the same responsibility spread too thin?** If yes, recommend consolidating
  or centralizing the related logic.
- **Does the module graph form a loop?** If yes, identify the cyclic dependency and
  suggest a clearer separation.
- **Does the code depend on hidden globals or state?** If yes, recommend explicit
  parameters or dependency injection.
- **Is reuse creating tight coupling?** If yes, recommend keeping related logic
  local until a stable abstraction emerges.
- **Is the flow easy to follow?** If no, recommend simplifying control paths,
  adding explicit branches, or clarifying intent.

---

## Quality criteria

A strong code-structure review should verify that the code is:

- **modular:** components have a single clear purpose,
- **loosely coupled:** dependencies are explicit and minimal,
- **cohesive:** related behavior lives together,
- **readable:** the flow is easy to understand at a glance,
- **testable:** units can be isolated and validated,
- **simple:** the design favors clarity over premature complexity,
- **explicit:** conditionals, error handling, and interfaces are well defined,
- **stable:** stable code does not depend on unstable or brittle modules.

---

## Example prompts

- "Review this code for structural issues and tell me whether it feels like
  spaghetti code."
- "Evaluate the module dependencies and identify any cycles or mesh-like
  coupling."
- "Does this implementation match the intended architecture, or has it become
  overly complex?"

---

## Related guidance

This skill complements:

- architecture-building-with-failure-in-mind
- architecture-risk-management
- project-management-adaptive-project-management
- business-analysis-scope