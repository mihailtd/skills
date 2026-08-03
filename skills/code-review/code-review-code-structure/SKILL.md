---
name: code-review-code-structure
description: Guides AI reviewers to evaluate code structure, detect architectural code smells, and recommend cleaner functional-lite designs — including flagging commented-out code, likely-dead/unused code, defensive validation duplicated across call sites, and feature flags that look fully settled in production for deletion.
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

8. **Flag commented-out code for deletion, not preservation**
   - Any commented-out block of code in a PR is a hard flag: recommend
     deletion, not "leave it in case we need it later." The codebase is
     under version control — the commit history is the correct place to
     recover an old version, not a comment block sitting in the current
     source.
   - This applies regardless of how the comment is phrased ("keeping this
     for reference," "TODO: remove," an entire disabled function) — the
     recommendation is the same: delete it, and trust `git log`/`git blame`
     to recover it if it's ever actually needed again.
   - Distinguish this from a genuine explanatory comment (documentation,
     a warning about a non-obvious constraint) — this point targets dead
     *code* left in comment form, not prose commentary.

9. **Flag dead/unused code as a heuristic finding, not an exhaustive guarantee**
   - Cross-reference a function, endpoint, or module's definition against
     its call sites; if nothing in the codebase still imports or calls it,
     flag it as a likely removal candidate. State this as a best-effort
     check (grep-based cross-referencing), not a proof of dead code —
     confirming genuine unreachability rigorously requires control-flow
     analysis, which is out of scope for a manual/line-level review.
   - A deprecated-but-never-removed endpoint or function is a
     particularly sharp case worth naming explicitly: it's a trap for a
     future engineer who finds it, assumes it's live, and builds on it.

10. **Flag defensive validation duplicated across many call sites**
    - Repeated inline null/empty/required-field checks scattered across
      many functions handling the same kind of object (common in
      dynamically-typed code without upfront schema validation) is a
      consolidation candidate — recommend a single validation
      function/model called from every site instead of duplicated checks.
    - Recommend the consolidation as a careful, file-by-file audit rather
      than a global find-and-replace: some of the "duplicate" checks may
      not actually be identical on inspection (an intentionally different
      constraint at one call site), and blindly collapsing them risks a
      real regression. In this codebase, the target shape is usually a
      Pydantic model validating the shape once at the boundary — see
      **python-fastapi-routing-validation** — rather than a hand-written
      `validate_x` helper threaded through every call site.

11. **Flag feature flags that look fully settled, not still in flux**
    - A feature-flag check (`if settings.FEATURE_X`, a flags/toggles
      config lookup) where the flag has clearly been fully enabled (or
      fully disabled) in production for a meaningful stretch, with no
      remaining variance in which path actually executes, should be
      flagged the same way as commented-out code — recommend removing the
      flag and collapsing to whichever path is actually live. Only make
      this recommendation once the flag's settled status is reasonably
      confirmed, not speculatively.

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
- **Is there commented-out code in the diff?** If yes, recommend deletion —
  never "leave it just in case." Version control already preserves it.
- **Is anything defined here never actually called elsewhere?** If so, flag
  it as a likely-dead-code candidate (heuristic, not proven), with a
  never-removed deprecated endpoint/function called out specifically.
- **Is the same validation logic duplicated across many call sites?** If
  yes, recommend consolidating into a single validation function/model
  (a Pydantic model at the boundary in this codebase) — but audit each
  site individually before collapsing them.
- **Does a feature flag in this diff look fully settled in production?** If
  so, recommend removing it and collapsing to the live path, once that
  settled status is actually confirmed.

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
- code-review-cognitive-load-smells
- code-review-linguistic-antipatterns
- python-fastapi-routing-validation (package `python-fastapi`) — the
  concrete Pydantic-at-the-boundary fix for point 10's defensive
  validation duplication.
- architecture-refactoring-decision-criteria (package `architecture`) —
  when dead code, duplicated validation, or a stale flag found here is
  extensive enough to warrant a dedicated refactoring effort rather than
  a PR-sized fix.