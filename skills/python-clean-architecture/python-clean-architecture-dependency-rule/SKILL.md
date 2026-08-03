---
name: python-clean-architecture-dependency-rule
description: Instructs the agent to enforce the Dependency Rule — source code dependencies point only inward, inner layers (domain/entities) know nothing about outer layers (infrastructure/interfaces) — using functional-lite modules (dataclasses + pure functions) organized by layer, not class hierarchies. Covers how this separation lets outer layers (CLI, web, database) be swapped without touching inner logic.
---

# Python Clean Architecture: The Dependency Rule

The Dependency Rule is the organizing principle of Clean/Onion
Architecture: source code dependencies must point only inward, toward
higher-level policy. Inner layers must know nothing about outer layers;
outer layers depend on and adapt to inner layers. This skill covers how to
enforce that rule in Python using functional-lite modules — plain functions
operating on immutable dataclasses, organized into layer packages — not the
class-hierarchy structure most Clean Architecture material illustrates.

---

## When to use this skill

Use this skill when you need to:

- lay out a new Python project or feature around Clean/Onion Architecture
  layers,
- decide which layer a piece of new code belongs in,
- review an import statement or module boundary for a Dependency Rule
  violation,
- explain why an inner layer must never import from an outer one, with a
  concrete example,
- swap an outer-layer implementation (e.g., CLI to web, SQLite to Postgres)
  and verify the inner layers didn't need to change.

---

## Outcome

Produce a layered structure that:

- organizes code into layer *packages* of plain functions and dataclasses
  (e.g., `domain/`, `application/`, `infrastructure/`, `interfaces/`), not
  layer *class hierarchies*,
- has inner-layer modules (`domain/`) that import nothing from outer-layer
  modules (`infrastructure/`, `interfaces/`) — ever,
- has outer-layer modules depend on and adapt to inner-layer functions and
  types, never the reverse,
- allows an outer layer to be replaced (a different UI, a different
  database) by changing only that layer's module, with zero changes
  required in `domain/` or `application/`,
- makes the Dependency Rule mechanically checkable, not just a design
  aspiration.

---

## Instructions for the AI

1. **Organize by layer package, not by class hierarchy**
   - Structure the codebase as layer packages, innermost to outermost —
     each corresponds to one ring of the traditional Clean Architecture
     diagram, reformulated as a package of dataclasses and functions
     rather than a package of classes:
     - `domain/` (**Entities** ring) — immutable dataclasses representing
       the core business nouns of the product: the objects that would
       exist even without any software (e.g., `Customer`, `Order`,
       `Task`), plus the pure functions expressing the most basic,
       universal rules about how they behave. No dependencies on anything
       outside `domain/`.
     - `application/` (**Use Cases** ring) — pure functions that
       orchestrate domain functions to accomplish one specific way the
       system is used. Recommend one function per use case, named for the
       scenario it represents (e.g., `create_task`, `complete_task`,
       `assign_task`), not for a technical operation — this keeps the
       layer readable as a list of "things the system does." Still free
       of I/O; may depend on `domain/`, nothing else.
     - `infrastructure/` (**Frameworks and Drivers** ring) — plain
       functions wrapping the actual external tools and delivery
       mechanisms the system runs on: the web framework (`fastapi` — the
       only backend framework this repo uses, see the `python` master
       skill), a database driver (SQLAlchemy async + `asyncpg` for the
       PostgreSQL path, Beanie or PyMongo's async driver for the MongoDB
       path), an email or payment client (`smtplib`, a payment SDK), or
       system utilities (logging, configuration). This is the most
       volatile layer — the one most likely to change as technology
       changes — which is exactly why it must stay outermost and
       replaceable. Depends on `domain/` (to know what shape of data to
       return) and possibly `application/`.
     - `interfaces/` (**Interface Adapters** ring) — the translation layer
       between `application/` and the outside world: functions that parse
       incoming requests (CLI args, HTTP request bodies) into calls on
       `application/` functions, and functions that format `application/`
       results for output (JSON responses, CLI text, rendered views).
       This is what lets the project decouple its use cases from whatever
       delivery mechanism currently exposes them. Depends on everything
       inward.
   - Note the two outermost rings serve different jobs even though both
     are "outer": `interfaces/` translates between use cases and the
     outside world; `infrastructure/` is the actual outside-world tooling
     being translated to/from. Keep them as separate packages rather than
     merging "everything not domain or application" into one folder — it
     keeps "what changed when we swapped databases" and "what changed
     when we added a web API" cleanly separable questions.
   - This is a package/module boundary, not a class boundary — there is no
     `Book` class with methods, no `BookInventory` service class, no
     `BookInterface` controller class. Each layer is a set of dataclasses
     and functions living in its own module or package.

2. **Lay out a concrete project structure, with tests mirroring the layers**
   - A representative functional-lite layout for a Clean Architecture
     Python project:
     ```
     src/
       entities/          # Entities ring — dataclasses + pure rules
         user.py
       use_cases/          # Use Cases ring — pure orchestration functions
         register_user.py
       interfaces/          # Interface Adapters ring — translators
         user_repository.py   # declares the Callable/NamedTuple "port"
         http_controllers.py
       frameworks/          # Frameworks & Drivers ring — real I/O
         database/
           orm.py            # concrete functions implementing the port
         web/
           app.py
     tests/
       entities/
       use_cases/
       interfaces/
       frameworks/
     ```
   - Mirror the `tests/` structure to the application structure, one test
     package per layer — this makes it immediately visible which layer a
     given test is exercising, and keeps entity/use-case tests
     (fast, no I/O) separate from frameworks tests (slower, may need real
     or containerized infrastructure).
   - This is where the dependency-inversion pattern (see
     python-clean-architecture-dependency-inversion) lives concretely: the
     "abstraction" — a `Callable` type alias or a `NamedTuple` of
     callables such as `UserRepository` — is *declared* in `interfaces/`
     (e.g., `interfaces/user_repository.py`), and *implemented* as plain
     functions in `frameworks/database/orm.py`. `use_cases/` imports the
     declared type from `interfaces/` to type its parameters, but never
     imports the concrete `frameworks/` implementation directly — that
     wiring happens only at the outermost entry point (e.g.,
     `frameworks/web/app.py`), where concrete functions are looked up and
     passed down into use-case calls.

3. **Reformulate class-based Clean Architecture examples into this shape**
   - Source material on Clean Architecture (books, articles) very commonly
     illustrates the pattern with classes — e.g., a `Book` entity class, a
     `BookInventory` class managing operations on books, and a
     `BookInterface` class handling user interaction. When translating
     such an example into this repo's style, reformulate it as:
     - `Book` → an immutable `@dataclass(frozen=True)` in `domain/`, with
       no methods — just fields.
     - `BookInventory`'s operations → free functions in `domain/` or
       `application/` that take a `Book` or `list[Book]` in and return a
       new `Book`/`list[Book]`/outcome out — e.g., `check_out(book:
       Book) -> Book`, `find_available(books: list[Book]) -> list[Book]`.
     - `BookInterface`'s responsibilities → free functions in
       `interfaces/` that parse input (CLI args, HTTP request bodies),
       call the `application/` functions, and format output — never a
       class instantiated to "handle" requests.
   - The behavioral guarantee described in such examples still holds
     exactly: functions in `domain/` know nothing about `interfaces/` or
     `infrastructure/`; functions in `application/` may use `domain/`
     functions but don't know about `interfaces/`. Only the *mechanism*
     changes — modules and functions instead of classes and methods.

4. **Enforce inward-only imports as a hard rule**
   - `domain/` modules must import only from the standard library, from
     `domain/` itself, or from genuinely framework-required third-party
     types (e.g., `pydantic.BaseModel` if used for a domain type's
     validation) — never from `application/`, `infrastructure/`, or
     `interfaces/`.
   - `application/` modules may import from `domain/`, but not from
     `infrastructure/` or `interfaces/`.
   - `infrastructure/` and `interfaces/` may import from any inward layer.
   - When reviewing a diff, check every new `import`/`from ... import` in
     an inner-layer file against this direction — a single inward-pointing
     violation (e.g., `domain/pricing.py` importing something from
     `infrastructure/db.py`) breaks the guarantee for the whole layer, not
     just that one file.
   - Watch specifically for standard-library convenience imports leaking
     into `domain/`/`application/` — Python's "batteries included" stdlib
     makes it tempting to `import smtplib` or similar directly inside a
     use case for convenience. Treat this the same as any other outward
     dependency: even a stdlib module represents a concrete implementation
     detail, so wrap it as a `Callable`-typed abstraction (see
     python-clean-architecture-dependency-inversion) declared in
     `interfaces/` and implemented in `frameworks/`/`infrastructure/`,
     rather than importing it directly in a use case just because it's
     "only stdlib."
   - Also watch for Python's flat, easy-import ecosystem quietly eroding
     the boundary over time — because every module is technically
     importable from anywhere, nothing stops a stray `from
     infrastructure.db import get_connection` from appearing inside
     `domain/` unless someone (or an automated check) is actively
     watching for it. Treat import-direction review as an ongoing
     vigilance task, not a one-time setup step.
   - Cross-reference python-architectural-fitness-functions for writing
     automated `ast`-based tests that catch these violations in CI, rather
     than relying on review alone.
   - Treat a dataclass field whose *type* is an infrastructure or
     framework concern as the sharpest, easiest-to-spot version of this
     violation — sharper than an import statement, because it shows up
     directly in the entity's own shape:
     ```python
     # Violation — the field's type alone breaks the Dependency Rule,
     # independent of whether the field is ever actually used:
     @dataclass
     class TaskWithDatabase:
         title: str
         db: DbConnection   # domain/ now depends on an infrastructure type
         ...

     @dataclass
     class ProjectWithUI:
         name: str
         ui: UiComponent    # domain/ now depends on a presentation type
         ...
     ```
   - The fix is to remove the field entirely — a `domain/` dataclass
     should have zero fields typed as a database connection, a UI
     component, an HTTP client, or any other infrastructure/framework
     type. Whatever that field was being used for (persisting the entity,
     refreshing a display) belongs in a use-case function in
     `application/` that receives the entity and calls a separately
     passed-in `Callable` to do the persisting/refreshing — see
     python-clean-architecture-dependency-inversion. Note this is also an
     SRP violation (see python-clean-architecture-single-responsibility):
     an entity holding a `db` field is now responsible for both being a
     domain concept and knowing how to persist itself.

5. **Use "can I swap the outer layer without touching the inner one" as the concrete test**
   - When validating that a design actually follows the Dependency Rule,
     pose the swap test directly: could the `interfaces/` layer be
     replaced entirely (e.g., a CLI script's functions swapped for a set
     of HTTP route handlers) without changing a single line in `domain/`
     or `application/`? Could `infrastructure/` be replaced (e.g., swap a
     SQLite-backed function for a Postgres-backed one with the same
     function signature) without changing `domain/` or `application/`?
   - If the answer to either is no, trace what's actually coupling the
     layers — usually an inner-layer function that quietly depends on a
     shape or behavior specific to one outer implementation — and fix the
     coupling rather than accepting the swap test failure as unavoidable.
   - Use this test as a design review question early (before writing code)
     and as a verification question later (after writing code), not just
     as an abstract principle.

6. **Explain the payoff concretely, not just as a principle**
   - Separation of concerns: each layer has one job and one reason to
     change.
   - Independence from external details: business rules in `domain/` and
     `application/` are completely unaffected by which database, web
     framework, or UI technology the project happens to use today.
   - Testability: `domain/` and `application/` functions can be
     unit-tested with plain data in, plain data out — no framework, no
     database, no mocks (see
     python-clean-architecture-functional-core-imperative-shell).
   - Maintainability: changing or replacing an outer-layer detail (a new
     UI, a new database) is a localized, low-risk change instead of a
     ripple through business logic.

---

## Decision points and guidance

- **Which layer package does this new function belong in?** Ask: does it
  make a business decision (→ `domain/`), orchestrate a use case without
  I/O (→ `application/`), perform actual I/O (→ `infrastructure/`), or
  handle external input/output formatting (→ `interfaces/`)?
- **Is a class being reached for here?** If the reference material or an
  existing pattern suggests a class (entity class, service class,
  controller class), stop and reformulate as a dataclass + functions in
  the appropriate layer package first.
- **Does this import point outward?** Any import from an inner-layer
  module reaching into an outer-layer module is a Dependency Rule
  violation — fix the direction, don't rationalize the exception.
- **Would swapping this outer-layer piece require inner-layer changes?**
  If yes, the coupling needs to be found and removed before considering
  the layering correct.

---

## Quality criteria

A strong Dependency-Rule implementation should ensure that:

- **layers are packages of functions and dataclasses**, not class
  hierarchies mirroring the traditional ring diagram,
- **`domain/` has zero outward dependencies**, verified by inspection or
  automated check,
- **`application/` depends only on `domain/`**, never on infrastructure or
  interface code,
- **outer layers are swappable in practice**, not just in theory — the
  swap test passes for both the interface and infrastructure layers,
- **class-based reference material is correctly translated** into
  functional-lite modules before being applied, never copied as-is.

---

## Example prompts

- "Lay out the package structure for this feature using Clean Architecture
  layers, functional-lite style."
- "This book example uses a `BookInventory` class — help me reformulate it
  as functions and dataclasses in the right layers."
- "Check this diff for any Dependency Rule violations — did anything in
  `domain/` end up importing from `infrastructure/`?"
- "Could we swap our CLI for a web API here without touching the domain or
  application layers? Let's verify."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-functional-core-imperative-shell
- python-clean-architecture-dependency-inversion
- python-clean-architecture-screaming-architecture
- python-clean-architecture-scaling
- python-clean-architecture-single-responsibility
- python-clean-architecture-aggregates
- python-clean-architecture-factories
- python-architectural-fitness-functions
- code-review-clean-architecture-boundaries
- python
