---
name: code-review-clean-architecture-boundaries
description: Guides AI reviewers to catch Clean Architecture Dependency Rule violations and layer-structure drift in a diff — an entity with an infrastructure-typed field, an inward layer importing outward, a controller/presenter doing more than translation, a "screaming" top-level structure that reveals its framework instead of its business purpose. Checks the layering itself, independent of whether the code is written in classes or functions (see code-review-clean-architecture-functional-style for that dimension).
---

# Code Review: Clean Architecture Boundaries

This skill helps AI reviewers catch violations of Clean Architecture's
Dependency Rule and layer structure in a Python diff — checking that
`domain/`/`application/` stay independent of `infrastructure/`/
`interfaces/`, that each layer does only the job that belongs to it, and
that new code lands in the right layer package in the first place. It
draws on the full `python-clean-architecture` skill package as the "what
correct looks like" baseline; this skill is the review-checklist
distillation of that package, not a replacement for it.

---

## When to use this skill

Use this skill when you need to:

- review a diff that touches `domain/`, `application/`, `infrastructure/`,
  or `interfaces/` in a Clean-Architecture-structured Python project,
- catch a new import, field, or dependency that crosses a layer boundary
  the wrong way,
- decide whether new code belongs where it was placed, or whether it's
  actually a different layer's concern,
- check that a controller, presenter, or use case is staying inside its
  own defined responsibility rather than absorbing another layer's job.

---

## Outcome

Produce a review that:

- flags any inward-layer (`domain/`, `application/`) code importing from,
  or depending on the type of, an outward layer (`infrastructure/`,
  `interfaces/`) — citing the specific import or field, not just "this
  feels wrong,"
- flags a dataclass field whose *type* is an infrastructure/framework
  concern (a DB connection, an HTTP client, a UI component) on a
  `domain/` entity — the sharpest, most concrete version of a Dependency
  Rule violation,
- checks that controllers only translate, use cases only orchestrate, and
  entities only hold business rules — flagging when one layer's code is
  quietly doing another layer's job,
- checks new top-level module/package names for the "screaming
  architecture" test — do they describe the business, or the framework,
  and does a new top-level folder actually belong, or drift the structure,
- treats an adapter that exists where no real format/protocol conversion
  is happening as unnecessary indirection, not automatically correct.

---

## Instructions for the AI

1. **Check import direction against the Dependency Rule**
   - For any new or changed import in `domain/` or `application/`, verify
     it points only to `domain/`, the standard library, or a genuinely
     framework-required third-party type (see
     python-clean-architecture-dependency-rule) — never to
     `infrastructure/` or `interfaces/`.
   - `application/` may import from `domain/`; it must not import from
     `infrastructure/` or `interfaces/`.
   - Treat one inward-pointing violation as breaking the guarantee for the
     whole layer, not just the one file — flag it accordingly, not as a
     minor nitpick.

2. **Check for infrastructure-typed fields on domain/application types**
   - Flag a dataclass field whose type is a database connection, an HTTP/
     API client, a UI component, or any other infrastructure concern, on
     anything defined in `domain/` or `application/` — this is a
     Dependency Rule violation visible directly in the type's own shape,
     independent of import statements (e.g., a `Task` entity with a `db:
     DbConnection` or `ui: UiComponent` field).
   - Recommend the fix: remove the field entirely; whatever it was used
     for belongs in a use-case function in `application/` that receives
     the entity and a separately-passed `Callable` to perform the actual
     I/O (see python-clean-architecture-dependency-inversion).

3. **Check that each layer's components stay inside their responsibility**
   - **Entities** (`domain/`): only business rules and invariants over the
     entity's own fields (see python-clean-architecture-entity-invariants).
     Flag entity-local code that reaches for I/O or another entity's data
     to make its decision, and flag entity-local code that performs a
     side effect (sending a notification, logging to an external system)
     rather than just computing a state transition.
   - **Use cases** (`application/`): orchestration only — flag business
     *decisions* embedded in what should be sequencing logic (see
     python-clean-architecture-use-cases), and flag a use case reaching
     directly for a concrete infrastructure implementation instead of
     receiving it as a parameter.
   - **Controllers** (`interfaces/`): translation only — flag a controller
     containing business logic, or importing a specific web/CLI framework
     directly rather than staying framework-agnostic (see
     python-clean-architecture-controllers).
   - **Presenters** (`interfaces/`): formatting only — flag a view
     containing formatting decisions that should have been computed by a
     presenter function instead (see python-clean-architecture-presenters'
     Humble Object pattern).

4. **Apply the "does this need an adapter at all" test**
   - When a new translation function/adapter is introduced at a layer
     boundary, check whether real format/protocol conversion is actually
     happening, or whether the outer implementation could satisfy the
     inner layer's `Callable` type directly with no translation needed
     (see python-clean-architecture-interface-adapters-boundary).
   - Flag an adapter that exists "for consistency" at a boundary that
     doesn't need one — it's indirection without a corresponding benefit.

5. **Check new structure against the Screaming Architecture test**
   - When a diff adds a new top-level module/package under `domain/` or
     `application/`, check whether its name describes the business
     concept/use case, or a technical/framework role (see
     python-clean-architecture-screaming-architecture) — flag `models.py`/
     `services.py`/`utils.py`-style naming as a missed opportunity to name
     things after what the system does.
   - Flag a *new top-level folder* outside the established layer
     hierarchy (e.g., a `notifications/` folder created next to `domain/`/
     `application/`/`infrastructure/`/`interfaces/` instead of inside
     `infrastructure/`) as architectural drift — this is exactly the kind
     of small, well-intentioned choice that erodes the structure over
     time if not caught immediately (see
     python-architectural-fitness-functions for automating this check).

6. **Weigh scale before insisting on full structure**
   - Before flagging a small project for lacking the full four-layer
     directory structure, check whether it's actually earned that
     structure yet (see python-clean-architecture-scaling) — a small
     script with a `logic.py`/`io.py` split may be correctly scaled, not
     under-structured. Reserve the full-structure expectation for projects
     that have genuinely grown into it.

---

## Decision points and guidance

- **Does this import/field point outward from an inner layer?** That's a
  Dependency Rule violation regardless of how small it looks.
- **Is a business decision embedded in what should be pure sequencing (a
  use case), or pure translation (a controller/presenter)?** Flag it and
  point to where it actually belongs.
- **Is a new top-level folder appearing outside the established layer
  hierarchy?** Treat that as drift to redirect, not a valid organizational
  choice.
- **Does this new adapter perform real conversion, or could the inner
  layer's type be satisfied directly?** Skip the adapter if it's not
  earning its keep.
- **Is the project's actual size and stage large enough to expect the full
  layer structure?** Don't flag intentionally minimal structure in a small
  project as a violation.

---

## Quality criteria

A strong Clean Architecture boundary review should confirm that:

- **no inward layer imports from, or types a field as, an outward layer**,
- **no entity or domain function performs I/O or a side effect**,
- **each layer's components stay inside their own responsibility** —
  entities decide, use cases sequence, controllers translate, presenters
  format,
- **adapters exist only where real format/protocol conversion happens**,
- **new top-level structure describes the business**, and doesn't
  introduce an unplanned folder outside the established layer hierarchy,
- **structural expectations are scaled to the project's actual size**.

---

## Review checklist

- [ ] Does every new/changed import in `domain/`/`application/` point only
      inward?
- [ ] Does any `domain/`/`application/` type have a field typed as an
      infrastructure/framework concern?
- [ ] Does an entity-local function perform I/O, reach into another
      entity, or trigger a side effect?
- [ ] Does a use case contain a business decision rather than pure
      sequencing?
- [ ] Does a controller contain business logic or import a specific
      framework directly?
- [ ] Does a presenter's corresponding view contain any formatting
      decision?
- [ ] Does a new adapter perform real format conversion, or is it
      unnecessary indirection?
- [ ] Does new top-level structure name the business, not the framework —
      and does it avoid introducing an unplanned top-level folder?
- [ ] Is the expected layer structure actually earned by this project's
      current size and stage?

---

## Example prompts

- "Review this diff for Clean Architecture Dependency Rule violations."
- "Does this new `Task` entity field violate our architectural
  boundaries?"
- "Someone added a `notifications/` folder at the project root — is that
  where it belongs?"
- "Is this new adapter actually needed, or could the driver satisfy the
  port directly?"

---

## Related guidance

This skill complements:

- code-review-clean-architecture-functional-style
- code-review-architecture
- code-review-detect-bad-design
- python-architectural-fitness-functions
