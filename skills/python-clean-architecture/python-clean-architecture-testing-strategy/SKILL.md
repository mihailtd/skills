---
name: python-clean-architecture-testing-strategy
description: Instructs the agent to concentrate test coverage where Clean Architecture's boundaries make it cheap — pure domain/use-case functions get exhaustive unit tests with zero setup, integration tests exercise one real boundary at a time (a real driver, with peripheral concerns stubbed) rather than the whole system, and end-to-end tests stay few. Test file structure mirrors the architecture's layer packages, making an awkward test import itself an early warning of a Dependency Rule violation.
---

# Python Clean Architecture: Testing Strategy

Clean Architecture's boundaries don't just organize production code — they
directly determine where testing effort pays off. This skill covers the
strategic side: how much of each test type to write and where, why an
awkward test is itself an architectural signal (already introduced in
python-clean-architecture-functional-core-imperative-shell, extended here
to the whole test suite), how to scope an integration test to exactly one
boundary, and how test file structure should mirror the layer packages
already established. See python-clean-architecture-test-doubles for the
mechanics of building test doubles for functional-lite code.

---

## When to use this skill

Use this skill when you need to:

- decide what proportion of unit, integration, and end-to-end tests a
  Clean Architecture Python project should have,
- scope a new integration test to the right boundary,
- organize a `tests/` directory for a project already following this
  package's layer structure,
- diagnose whether a hard-to-write test is revealing an actual
  architecture problem,
- explain why Clean Architecture reduces reliance on slow, brittle
  end-to-end tests.

---

## Outcome

Produce a test suite that:

- concentrates the bulk of its coverage in fast, zero-setup unit tests
  against pure `domain/`/`application/` functions — the natural result of
  those functions already being pure (see
  python-clean-architecture-functional-core-imperative-shell),
  reserves a smaller set of integration tests for verifying one concrete
  boundary at a time (a real repository against a temp directory, with
  everything else stubbed),
- keeps end-to-end tests few, reserved for the handful of workflows that
  genuinely need to verify the whole system wired together,
- mirrors `tests/` structure to the `domain/`/`application/`/
  `infrastructure/`/`interfaces/` layout already established, so an
  awkward test import is itself a visible Dependency Rule warning,
- treats test-writing friction (needing to reach for a mocking library,
  needing multi-step setup for a supposedly simple test) as a prompt to
  reconsider the production code, not just the test.

---

## Instructions for the AI

1. **Weight the test suite toward fast, pure unit tests**
   - Because `domain/` and `application/` functions are pure (immutable
     dataclasses in, immutable dataclasses/`Result` out — see
     python-clean-architecture-functional-core-imperative-shell and
     python-clean-architecture-entity-invariants), they should make up the
     large majority of the test suite: each test constructs plain input
     data, calls the function, asserts on plain output data, with no
     database, no filesystem, no network, and no test double library
     needed.
   - Treat a `domain/`/`application/` test that reaches for `Mock`,
     `monkeypatch`, or any setup beyond constructing dataclasses as a
     signal something impure has leaked into what should be a pure
     function — fix the production code's core/shell boundary before
     adding more test infrastructure to work around it.

2. **Scope integration tests to exactly one boundary**
   - An integration test exists to verify something a unit test
     structurally cannot: that a *concrete* driver (see
     python-clean-architecture-drivers) actually does what its `Callable`
     type promises — e.g., that a file-backed repository's `get`/`save`
     pair round-trips data correctly through real disk I/O.
   - Test the real implementation for the one boundary being verified, and
     use simple stand-in functions (see python-clean-architecture-test-doubles)
     for every other dependency in the same test — e.g., testing task
     creation with a real file-backed repository pair while a
     notification function is a no-op stand-in, since the notification
     behavior isn't what this test is about.
   - Resist testing every combination of real components together — that
     drifts toward end-to-end testing and loses the focus that makes
     integration tests fast and diagnostic. One test, one boundary.

3. **Reserve end-to-end tests for the few workflows that need them**
   - Keep end-to-end tests (driving the actual CLI or HTTP interface
     start-to-finish) to a small number covering genuinely critical user
     workflows — the top of the testing pyramid, not its base.
   - Rely on unit and integration test coverage to catch the vast majority
     of regressions; treat a growing reliance on end-to-end tests to catch
     ordinary bugs as a sign that unit/integration coverage has gaps
     worth closing instead.

4. **Mirror `tests/` structure to the architecture's layer packages**
   - Structure the test directory to match the production layout already
     established (see python-clean-architecture-dependency-rule):
     ```
     tests/
       domain/
         test_task.py
         test_deadline.py
       application/
         test_create_task_use_case.py
       infrastructure/
         test_file_task_repository.py
       interfaces/
         test_task_controller.py
     ```
   - This isn't just organizational tidiness — it makes the Dependency
     Rule visible through the test suite too: a test under `tests/domain/`
     that needs to import something from `infrastructure/` should feel
     immediately wrong, the same way the equivalent production import
     would (see python-architectural-fitness-functions for automating this
     check).

5. **Treat test-writing friction as architectural feedback**
   - When a test is hard to write — needs elaborate setup, needs several
     unrelated things stubbed just to exercise one behavior, or is
     unclear about what specifically it's verifying when it fails — treat
     that friction as a direct signal about the production code, not just
     an inconvenience to engineer around.
   - This extends the diagnostic already established for the core/shell
     boundary specifically (python-clean-architecture-functional-core-imperative-shell)
     to the whole suite: a hard-to-test controller usually means it's
     doing too much (see python-clean-architecture-single-responsibility);
     a hard-to-test use case usually means a dependency wasn't properly
     inverted (see python-clean-architecture-dependency-inversion).

---

## Decision points and guidance

- **Is this a unit test, an integration test, or does it belong at the
  end-to-end level?** Default to unit; reach for integration only to
  verify a specific driver actually satisfies its `Callable` type against
  something real; reserve end-to-end for a small set of critical
  workflows.
- **Does this integration test touch more than one real boundary at
  once?** Narrow it — stub everything except the one boundary actually
  being verified.
- **Does a test's file location match where the code it's testing
  lives?** If not, realign `tests/` to mirror the layer structure.
- **Is this test hard to write?** Ask what that difficulty is revealing
  about the production code before adding more test scaffolding to push
  through it.

---

## Quality criteria

A strong Clean Architecture test suite should ensure that:

- **the bulk of coverage is fast, pure unit tests** requiring no test
  doubles beyond plain data construction,
- **each integration test verifies exactly one real boundary**, with
  everything else stubbed,
- **end-to-end tests stay few**, reserved for genuinely critical
  workflows,
- **`tests/` mirrors the production layer structure**, making Dependency
  Rule violations visible through import paths in tests too,
- **test-writing friction is treated as a design signal**, prompting a
  look at the production code's boundaries, not just more test
  infrastructure.

---

## Example prompts

- "How should we distribute our test coverage across unit, integration,
  and end-to-end tests for this Clean Architecture project?"
- "This integration test touches three real repositories at once — help
  me narrow it to the one boundary it's actually supposed to verify."
- "This use case is really hard to test — what does that tell us about
  how it's structured?"

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-test-doubles
- python-clean-architecture-functional-core-imperative-shell
- python-clean-architecture-drivers
- python-testing-mocking
- python-architectural-fitness-functions
- python-clean-architecture-incremental-migration
