---
name: architecture-inheritance-coupling-tradeoffs

description: Instructs the AI assistant to treat inheritance-based deduplication — extracting a shared base class or algorithm from near-duplicate components — as trading duplication for tight coupling to that shared implementation, since every dependent becomes bound to its exact behavior; recommends verifying the shared logic is genuinely identical and stable before extracting, and preferring composition over inheritance when it isn't.
---

# Inheritance Coupling Tradeoffs Instructions

When supporting architecture or code-level design, use this tool to extend the
duplication-vs-coordination tradeoff down to the code level: extracting a
shared base class or algorithm from near-duplicate components removes
duplication but creates tight coupling between every dependent and that shared
implementation — treat this as the same tradeoff covered in
architecture-code-duplication-tradeoffs, now paid inside a single codebase
instead of across team boundaries.

---

## Purpose

This tool helps the AI assistant by:

- treating structural similarity between two components as necessary but not
  sufficient justification for a shared abstraction — the shared behavior must
  be genuinely identical for all current and reasonably foreseeable callers,
  not just similar-looking today,
- explaining that inheritance-based sharing doesn't eliminate the coordination
  cost of duplicated code, it relocates it — from "coordinate across teams or
  codebases" to "coordinate across every dependent of a shared parent class,"
- recommending composition (a small, focused, independently callable unit of
  shared behavior) as the lower-coupling default, reserving inheritance for
  cases where the shared process and its variation points are both genuinely
  stable,
- recognizing a growing number of overridden methods or type-branching
  conditionals inside a shared parent as a concrete signal that an abstraction
  has outgrown inheritance and should be unwound.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- distinguishes what's actually identical between two components (the
  algorithm or process) from what merely varies through a single, well-isolated
  hook, before recommending a shared abstraction,
- warns that once several components inherit shared behavior, a change made to
  satisfy one dependent's new requirement risks silently changing behavior for
  every other dependent too,
- defaults to composition/delegation over inheritance whenever the variation
  between components is more than one clearly isolated method,
- keeps duplicated-but-independent implementations as the right call for
  components whose similarity today is coincidental rather than a genuinely
  shared responsibility.

---

## Instructions for the AI

1. **Verify the duplication is genuine, not coincidental**
   - Two components that happen to look alike today (the same overall shape,
     a similar field layout) aren't necessarily solving the same problem — ask
     whether they're expected to evolve together or independently before
     treating the similarity as something worth extracting.
   - A shared abstraction earns its cost only when the *entire* algorithm,
     minus a small number of well-defined variation points (e.g., a single
     "how to format this value" hook), is truly identical — not merely "mostly
     the same for now, with the rest to be reconciled later."

2. **Recognize the coupling inheritance introduces**
   - Every dependent of a shared base becomes bound to that base's exact
     implementation — a bug fix, a performance change, or a new requirement in
     the shared logic now needs to be evaluated against every dependent's
     behavior, not just the one that prompted the change.
   - Name this explicitly as the same synchronization tax covered in
     architecture-code-duplication-tradeoffs, just relocated: inheritance
     doesn't remove the coordination cost of shared code, it moves it from
     "coordinate between teams" to "coordinate across every caller of the
     shared class" — and a reviewer now needs to check every dependent, not
     only the one being changed, before approving a change to the shared base.

3. **Prefer composition over inheritance for shared behavior**
   - Recommend extracting genuinely identical logic as a small, focused,
     independently callable unit that each component uses (composition or
     delegation), rather than a base class each component extends — this
     scopes the coupling to "calls this one function or interface" instead of
     "extends this class's entire behavior and internal assumptions."
   - Reserve inheritance specifically for cases where the variation between
     dependents really is a single, clean, unlikely-to-multiply extension
     point, and the shared parent's behavior is stable and unlikely to need
     per-caller special-casing later.
   - Treat a growing number of overridden methods, or conditional logic inside
     the shared parent that branches by dependent type, as a signal the
     abstraction has already outgrown inheritance — recommend unwinding it
     before adding more dependents on top of it, not patching around the
     branching.

4. **Re-evaluate independent duplication as a legitimate fallback**
   - If two "duplicate" components' likely evolution paths actually diverge
     (one needs a feature the other never will), recommend keeping them
     duplicated and independent rather than forcing a shared abstraction that
     will need to special-case one caller almost immediately.
   - Apply the same evaluation as architecture-code-duplication-tradeoffs at
     this granularity: is the real maintenance cost of occasional drift
     between two small, independent pieces of logic actually higher than the
     coupling cost of forcing them through one shared implementation? Don't
     assume the answer is yes by default.

---

## AI decision guidance

When generating inheritance/coupling guidance, keep these principles in mind:

- **Structural similarity today is not sufficient justification for a shared
  base class** — check whether the components are expected to evolve
  together before extracting.
- **Inheritance-based sharing relocates the coordination cost of duplication,
  it doesn't eliminate it** — from across teams/codebases to across every
  dependent of the shared parent.
- **Composition is the lower-coupling default**; inheritance is justified only
  for a genuinely stable shared process with one clean variation point.
- **Growing overrides or type-branching inside a shared parent is a concrete
  signal to unwind the abstraction**, not a reason to keep patching it.

---

## Success criteria

A strong response should ensure that it:

- **checks whether the duplication is coincidental similarity or a genuinely
  shared responsibility** before recommending extraction,
- **names the specific coupling cost introduced** — every dependent now relies
  on the shared implementation's exact details,
- **defaults to composition over inheritance** unless the stability and
  single-variation-point conditions are clearly met,
- **treats a growing number of dependent-specific special cases** inside a
  shared base as a signal to revisit the abstraction, not extend it further.

---

## Example prompts for the AI

- "We have two request handlers that are almost identical except for one
  formatting method — should we share a base class?"
- "Our shared base class keeps growing new if-else branches for different
  subclasses — is that a sign of a problem?"
- "Should we use inheritance here, or just call a shared helper function from
  both components?"

---

## Related guidance

Use this tool alongside:

- architecture-code-duplication-tradeoffs
- architecture-decomposition
- code-review-detect-bad-design (package `code-review`) — functional-lite composition and coupling smells at review time
- architecture-exception-design-and-anti-patterns — the same source book's guidance on exception-handling design
- architecture-flexibility-complexity-tradeoffs — the same source book's flexibility-vs-complexity axis, applied to API extensibility rather than code-level duplication
- architecture-configuration-surface-tradeoffs — the same axis again, applied to a dependency's configuration surface
