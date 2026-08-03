---
name: python-clean-architecture-domain-modeling
description: Instructs the agent to apply Domain-Driven Design's tactical patterns (entities, value objects, domain services, bounded contexts, ubiquitous language) in functional-lite Python — entities are frozen dataclasses with an id field and identity checked via a dedicated function (never an overridden __eq__/__hash__ on a mutable base class), value objects are frozen dataclasses with default structural equality (a natural, no-reformulation fit), and domain services are just plain functions, never stateless "service" classes.
---

# Python Clean Architecture: Domain Modeling with DDD, Functional-Lite

Domain-Driven Design (DDD) gives Clean Architecture's Entity/Domain layer its
tactical patterns: entities, value objects, domain services, bounded
contexts, and a shared ubiquitous language. DDD reference material almost
always implements these as classes — a mutable `Entity` base class with
overridden `__eq__`/`__hash__`, service classes with one stateless method.
In functional-lite, each pattern gets a specific, often *simpler*
reformulation: value objects and domain services translate to functional
style almost for free; entities need one deliberate design decision (how to
check identity without breaking structural equality for tests).

---

## When to use this skill

Use this skill when you need to:

- model a new domain concept and decide whether it's an entity, a value
  object, or logic that belongs in a domain service,
- translate a DDD reference example (entity base class, service class) into
  this repo's style,
- establish a shared vocabulary between code and business/domain experts
  for a new feature area,
- decide where a business rule belongs — inside an entity's own module, or
  in a domain service / use case — when it spans more than one entity,
- organize a system into bounded contexts as it grows.

---

## Outcome

Produce a domain model that:

- represents entities as frozen dataclasses with an `id` field, using a
  dedicated function to check "is this the same real-world thing" rather
  than overriding `__eq__`/`__hash__` — so the dataclass's default
  structural equality stays available and meaningful for tests,
- represents value objects as frozen dataclasses relying on default
  (structural) equality — the natural, already-idiomatic Python mapping,
  requiring no special-casing at all,
- represents domain services as plain functions, never as classes with a
  single stateless method,
- organizes bounded contexts as separate packages/modules, each with its
  own domain vocabulary, rather than as separate class namespaces,
- is grounded in a consistent, business-aligned vocabulary applied the same
  way in code, tests, and conversation with domain experts.

---

## Instructions for the AI

1. **Resist writing code before the domain is understood**
   - Before modeling anything, work through the domain's defining
     questions directly with whoever knows the business: what makes each
     concept unique, what states/transitions are valid, what relationships
     hold between concepts, what happens at edge cases (a deadline
     passing, a status changing).
   - Treat the urge to start coding immediately as a signal to slow down —
     the investment in understanding the domain before modeling it pays
     off in a model that actually fits, rather than one that has to be
     reshaped repeatedly as gaps in understanding surface later.

2. **Establish and reuse a ubiquitous language**
   - Fix a specific, shared vocabulary for the domain's concepts (e.g.,
     Task, Project, Due Date, Priority, Status) and use those exact terms
     consistently across code (module names, dataclass field names,
     function names), tests, and conversation — never introduce a code-only
     synonym for a term the business already has a name for.
   - This is the same discipline as this repo's naming-consistency
     practice applied one level up, from "consistent names within a
     codebase" to "consistent names between the codebase and the business"
     — see code-review-naming-consistency for the concrete mechanics (name
     molds, a project lexicon) that keep this vocabulary from drifting.

3. **Model entities as frozen dataclasses with a dedicated identity check**
   - Give every entity an `id` field, but represent the entity itself as an
     immutable, frozen dataclass — not a mutable class inheriting from a
     shared `Entity` base class:
     ```python
     from dataclasses import dataclass, field
     from uuid import UUID, uuid4

     @dataclass(frozen=True)
     class Task:
         id: UUID = field(default_factory=uuid4)
         title: str = ""
         description: str = ""
         # ... other fields ...
     ```
   - Don't reach for a shared `Entity` base class purely to get the `id`
     field — repeating `id: UUID = field(default_factory=uuid4)` in each
     entity dataclass is cheap, explicit, and avoids introducing a
     hierarchy to reason about. (If a team genuinely wants to avoid
     repeating the field declaration across many entity types, a frozen
     dataclass used purely for field composition — carrying zero methods —
     is an acceptable lightweight exception; the moment any method is
     added to it, stop and reformulate as plain functions instead.)
   - **Do not override `__eq__`/`__hash__`** to make equality identity-based.
     This is the key departure from the classic DDD implementation, and
     it's a genuine improvement, not just an OOP-avoidance workaround: if
     `__eq__` is overridden to compare only `id`, then `assert result ==
     expected_task` in a test will pass even when every business-relevant
     field is wrong (as long as the UUIDs happen to match), and will fail
     when everything *except* the auto-generated UUID matches — both
     surprising outcomes for anyone reading the test. Leave the dataclass's
     default, all-fields structural equality in place; it's exactly what a
     test asserting "this is the Task I expected" should use.
   - Check identity — "is this the same real-world entity, possibly with
     different attribute values" — with a small, explicit function instead:
     ```python
     def same_task(a: Task, b: Task) -> bool:
         return a.id == b.id
     ```
     Use this function specifically where identity (not full structural
     equality) is the actual question — e.g., deduplicating a list of task
     snapshots, or checking whether an update targets an existing task.

4. **Model value objects as frozen dataclasses — the natural fit**
   - Value objects (things defined by their attributes, not an identity —
     a `Deadline`, a `Money` amount) map to frozen dataclasses with no
     special handling at all: default structural equality is exactly
     right, since two value objects with the same attributes genuinely
     *are* equal.
   - Use a value object (an `Enum`, or a small frozen dataclass) anywhere a
     primitive would otherwise carry domain meaning — this prevents
     "primitive obsession" and buys real type safety `ty` can check:
     ```python
     # Primitive obsession — compiles, but allows nonsense and silent bugs
     task.status = "Finished"           # any string is "valid"
     task.status == "done"              # False — case mismatch, no error

     # Value object — invalid values are unrepresentable
     class TaskStatus(Enum):
         TODO = "TODO"
         IN_PROGRESS = "IN_PROGRESS"
         DONE = "DONE"

     task = replace(task, status=TaskStatus.DONE)  # ty flags a typo'd value
     task.status == TaskStatus.DONE                 # True, unambiguous
     ```
   - This is the same instinct as `NewType` for distinct identifiers (see
     python-core's python-data-structures-type-system) applied to a closed
     set of valid values instead of a single wrapped primitive.
     ```python
     from dataclasses import dataclass
     from datetime import datetime, timedelta, timezone

     @dataclass(frozen=True)
     class Deadline:
         due_date: datetime

         def __post_init__(self) -> None:
             if self.due_date < datetime.now(timezone.utc):
                 raise ValueError("Deadline cannot be in the past")
     ```
   - Validation in `__post_init__` is already idiomatic and requires no
     reformulation — it runs once at construction, doesn't mutate
     anything, and rejects invalid states before they can exist. This is
     exactly the discipline functional-lite wants: invalid states are
     unrepresentable, enforced at the single point of construction.
   - Read-only computations that only inspect `self` (no mutation) are
     acceptable as methods on a frozen value object *or* as free functions
     taking the value object as a parameter — both are behaviorally
     identical since there's no mutation risk either way. Prefer free
     functions for consistency with the rest of the codebase's style, but
     don't treat a pure read-only method on a frozen dataclass as a
     violation if it's more ergonomic at a given call site:
     ```python
     def is_overdue(deadline: Deadline) -> bool:
         return datetime.now(timezone.utc) > deadline.due_date

     def time_remaining(deadline: Deadline) -> timedelta:
         return max(timedelta(0), deadline.due_date - datetime.now(timezone.utc))
     ```

5. **Model domain services as plain functions, never as classes**
   - A "domain service" in classic DDD is stateless logic that spans
     multiple entities/value objects and doesn't naturally belong to any
     one of them (calculating a task's effective priority from several
     factors, deciding when to send a reminder). In functional-lite, this
     is simply a function — there is no reformulation step needed beyond
     "don't wrap it in a class":
     ```python
     def calculate_task_priority(task: Task, related_tasks: list[Task]) -> Priority:
         ...

     def should_send_reminder(task: Task, now: datetime) -> bool:
         ...
     ```
   - If reference material shows a `TaskPriorityCalculator` or
     `ReminderService` class with one public method and no meaningful
     internal state, that class *is* the function — translate it directly
     by dropping the class wrapper, not by finding a "functional
     equivalent" of it (there's nothing left to equivalence-map once the
     wrapper is removed).

6. **Organize bounded contexts as packages, not class namespaces**
   - A bounded context (e.g., Task Management, User Account Management,
     Notifications) becomes a top-level package containing its own
     domain/application modules (see python-clean-architecture-dependency-
     rule for the layer-package structure within each context) — not a
     namespace created via class nesting or a shared abstract base.
   - Each bounded context can use the same term differently if the
     business genuinely does (e.g., "User" might mean something narrower
     in Notifications than in User Account Management) — the package
     boundary is what makes this safe; keep the ubiquitous language
     consistent *within* a bounded context, and expect it to legitimately
     vary *across* contexts.

---

## Decision points and guidance

- **Does this concept have a persistent identity, or is it defined by its
  attributes?** Identity → entity (frozen dataclass + `id` field +
  dedicated identity-check function). Attributes → value object (frozen
  dataclass, default equality).
- **Is this logic naturally owned by one entity, or does it span several?**
  One entity → a function living alongside that entity's module. Several →
  a domain service function, or a use-case function in `application/` if
  it also needs orchestration/I/O.
- **Is `__eq__`/`__hash__` being overridden on an entity?** Stop — use a
  dedicated identity-check function instead, and leave default structural
  equality in place for tests.
- **Is a "service" class being defined with one method and no real
  state?** Drop the class; it's a function.
- **Does a term mean different things in different parts of the system?**
  That's a signal for separate bounded-context packages, not a sign the
  vocabulary needs to be forced into agreement everywhere.

---

## Quality criteria

A strong functional-lite domain model should ensure that:

- **entities are frozen dataclasses with an `id` field**, with identity
  checked via a dedicated function, never via overridden `__eq__`/`__hash__`,
- **structural equality (`==`) stays meaningful for tests** on every entity,
- **value objects use default structural equality** with no special-casing,
- **validation happens in `__post_init__`**, rejecting invalid states at
  construction rather than allowing them to exist and be checked later,
- **domain services are plain functions**, with no stateless "service"
  class wrapping them,
- **bounded contexts are packages**, each with an internally consistent
  vocabulary that's allowed to differ from other contexts' usage of the
  same term.

---

## Example prompts

- "This DDD example has an `Entity` base class with overridden `__eq__` —
  reformulate it so tests can still compare Task snapshots structurally."
- "Is 'Priority' here an entity or a value object, and how should I model
  it?"
- "This `ReminderService` class has one method and no state — just make it
  a function."
- "We're splitting into bounded contexts — how should that map to our
  package structure?"

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-entity-invariants
- python-clean-architecture-aggregates
- python-clean-architecture-factories
- python-clean-architecture-dependency-rule
- python-clean-architecture-single-responsibility
- python-clean-architecture-legacy-assessment
- code-review-naming-consistency
