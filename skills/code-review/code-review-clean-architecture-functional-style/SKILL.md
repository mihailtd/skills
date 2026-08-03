---
name: code-review-clean-architecture-functional-style
description: Guides AI reviewers to catch OOP creep specifically in Clean Architecture code — a "use case"/"controller"/"service" class with one method instead of a function, an ABC where a Callable type alias belongs, a mutating entity method instead of a pure transition function, Mock(spec=SomeABC) in a test where a stand-in function belongs. This is the house-style dimension of Clean Architecture review — a diff can nail the Dependency Rule while still being full of the class-based patterns this repo doesn't use.
---

# Code Review: Clean Architecture Functional Style

A diff can respect Clean Architecture's layering perfectly and still
violate this repo's house style by reaching for classes, inheritance, and
mutation where this codebase uses dataclasses, closures, and pure
functions instead. This skill is the review-checklist companion to
python-clean-architecture-dependency-inversion,
python-clean-architecture-entity-invariants, and
python-clean-architecture-drivers — checking specifically for the
class-based patterns those skills reformulate, so they get caught in
review even when the underlying layering is otherwise correct.

---

## When to use this skill

Use this skill when you need to:

- review a diff in a Clean-Architecture-structured Python project for
  unnecessary classes, inheritance, or mutation,
- catch an ABC-based "port" where a `Callable` type alias belongs,
- catch a use-case/controller/presenter implemented as a class with one
  method instead of a plain function,
- catch an entity method that mutates `self` instead of returning a new
  value,
- catch a test reaching for `Mock(spec=SomeABC)` against a dependency that
  should just be a stand-in function.

---

## Outcome

Produce a review that:

- flags any `ABC`/`abstractmethod` used to define a dependency's contract,
  recommending a `Callable` type alias (or a `NamedTuple`/frozen dataclass
  bundle of callables) instead,
- flags a "use case"/"controller"/"presenter"/"factory" class holding
  constructor-injected dependencies with one public method, recommending a
  plain function taking those dependencies as parameters,
- flags an entity or value-object method that mutates `self` (`self.status
  = ...`, `self._items.append(...)`) instead of returning a new value via
  `dataclasses.replace`,
- flags a driver/adapter class implementing an ABC, recommending the
  closure-based factory-function pattern instead,
- flags `Mock()`/`Mock(spec=...)` used against a repository- or
  service-style dependency in a test, recommending a plain stand-in
  function.

---

## Instructions for the AI

1. **Flag ABC-based ports; recommend `Callable` type aliases**
   - Any `class SomethingPort(ABC)` (or a `Protocol` with a single method
     and concrete classes implementing it) defining a swappable dependency
     is a candidate for a `Callable[[...], ...]` type alias instead — see
     python-clean-architecture-dependency-inversion.
   - Reserve acceptance of `Protocol` only for the narrow case of typing a
     *bundle* of several related callables at a public boundary — flag it
     if it's standing in for a single-operation interface, or for
     anything with real attached behavior.
   - `ABC` is never appropriate for a business-logic dependency's contract
     in this repo — flag every instance, not just the more obviously
     awkward ones.

2. **Flag interactor-shaped classes; recommend plain functions**
   - A class whose entire shape is "constructor-injected dependencies as
     fields, one public method that does the real work" — a use case
     (`execute`), a controller (`handle_x`), a presenter (`present_x`), or
     a factory — is a function wearing a class's clothing. Flag it and
     recommend the direct function form: dependencies as parameters,
     calling the function with them supplied *is* the injection (see
     python-clean-architecture-use-cases,
     python-clean-architecture-controllers,
     python-clean-architecture-presenters,
     python-clean-architecture-factories).
   - This applies even when the class is a `@dataclass` — a frozen
     dataclass with fields plus one meaningful method is still this
     pattern; frozen alone doesn't make it correct.

3. **Flag mutating entity/aggregate methods; recommend pure transition functions**
   - Any method that assigns to `self.<field>` or mutates a mutable field
     in place (`self._tasks[id] = task`, `self.items.append(...)`) on a
     domain entity or aggregate is flagged — recommend a plain function
     taking the current (immutable) value and returning a new one via
     `dataclasses.replace` (see python-clean-architecture-entity-invariants
     and python-clean-architecture-aggregates).
   - Check that the aggregate's own collection field is an immutable type
     (`tuple`, not `list`/`dict`) — a `dict`/`list` field on an aggregate
     is a strong signal it's about to be mutated in place somewhere.
   - Flag an entity-local method that also performs a side effect
     (sending a notification, writing a log entry) alongside its state
     change — the side effect belongs in the use case that calls the pure
     transition function, sequenced explicitly.

4. **Flag driver/adapter classes; recommend closures**
   - A class implementing an ABC to wrap a database, file, or third-party
     API client is flagged — recommend a factory function that creates
     the needed private state once and returns plain functions closing
     over it, matching the port's `Callable` type (see
     python-clean-architecture-drivers).
   - Check that configuration needed by the driver is passed into the
     factory function as parameters, not read internally via a `Config`
     class or `os.getenv` calls scattered inside the driver.

5. **Flag `Mock(spec=SomeABC)` in tests against repository/service dependencies**
   - Once ports are `Callable` types (per step 1), there's no ABC left to
     `spec=` against — flag `Mock()`/`Mock(spec=...)` used for a
     repository or service dependency and recommend a plain stand-in
     function instead: a lambda for a fixed return, a small named function
     for conditional behavior, or a recording closure (`list.append`) for
     verifying what was called (see python-clean-architecture-test-doubles).
   - Don't flag `Mock`/`monkeypatch` used for genuinely unpredictable
     module-level things (`datetime.now()`, `random`, filesystem/OS calls)
     — that usage is correct per python-testing-mocking; the distinction
     is what's being mocked, not the tool itself.

6. **Recognize the legitimate exceptions — don't over-flag**
   - Don't flag classes required by a framework's own extension mechanism
     (a `logging.Formatter` subclass, a SQLAlchemy `DeclarativeBase`
     model, a Pydantic `BaseModel`, a Beanie `Document`, a custom
     `Exception` subclass, test classes) — these are the established,
     legitimate exceptions (see the `python` master skill's "reserve real
     OOP classes for cases the framework requires" rule).
   - Don't flag a `@classmethod` alternative constructor or a pure,
     read-only method on a frozen dataclass (no mutation involved) as
     urgent — these are acceptable, lower-priority style choices per
     python-clean-architecture-factories and
     python-clean-architecture-domain-modeling, not violations on the
     level of mutation or ABC-based ports.

---

## Decision points and guidance

- **Is a dependency's contract defined with `ABC`/`abstractmethod`?**
  Flag it — recommend a `Callable` type alias.
- **Does a class have injected dependencies as fields and one real
  method?** Flag it — recommend a plain function with those dependencies
  as parameters.
- **Does a method assign to `self.<field>` or mutate a mutable field in
  place?** Flag it — recommend a pure function returning a new value via
  `dataclasses.replace`.
- **Is a repository/service driver implemented as a class implementing an
  ABC?** Flag it — recommend a closure-returning factory function.
- **Is `Mock(spec=...)` used against a repository/service dependency in a
  test?** Flag it — recommend a stand-in function. Is it used against a
  genuinely unpredictable module-level thing? That's fine, leave it.
- **Is the class in question a framework-mandated extension point** (ORM
  model, Pydantic model, `logging.Formatter`, exception class, test
  class)? Don't flag it.

---

## Quality criteria

A strong functional-lite Clean Architecture review should confirm that:

- **no dependency contract is defined with `ABC`**, only `Callable` types
  (or narrowly-scoped `Protocol`/`NamedTuple` bundles),
- **no use case, controller, presenter, or factory is a class** with
  injected fields and one method,
- **no entity or aggregate method mutates `self`** — all state changes are
  pure functions returning new values,
- **no driver/adapter is a class implementing an ABC** — all are
  closure-returning factory functions,
- **no test uses `Mock(spec=...)` against a repository/service
  dependency** — only against genuinely unpredictable module-level
  behavior,
- **legitimate framework-mandated classes are left alone**, not flagged
  just for being classes.

---

## Review checklist

- [ ] Does any dependency contract use `ABC`/`abstractmethod` instead of a
      `Callable` type?
- [ ] Does any use case/controller/presenter/factory take the shape of a
      class with injected fields and one method?
- [ ] Does any entity/aggregate method assign to `self.<field>` or mutate
      a mutable field in place?
- [ ] Is an aggregate's collection field a mutable type (`list`/`dict`)
      instead of an immutable one (`tuple`)?
- [ ] Does any driver/adapter implement an ABC as a class instead of being
      a closure-returning factory function?
- [ ] Is driver configuration read internally (`Config`/`os.getenv`)
      instead of passed in as parameters?
- [ ] Does any test use `Mock(spec=...)` against a repository/service
      dependency rather than a stand-in function?
- [ ] Are all flagged classes genuinely not one of the framework-mandated
      exceptions?

---

## Example prompts

- "Review this diff for classes that should be plain functions instead."
- "This entity method mutates `self.status` — flag it and show the pure
  version."
- "Is this `Mock(spec=TaskRepository)` in our test suite still valid, or
  does it need to be a stand-in function now?"
- "Check this driver implementation for unnecessary class/inheritance
  usage."

---

## Related guidance

This skill complements:

- code-review-clean-architecture-boundaries
- code-review-detect-bad-design
- code-review-quality-and-hygiene
