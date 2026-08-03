---
name: python-clean-architecture-aggregates
description: Instructs the agent to implement DDD aggregates (a root entity plus its owned children, treated as one consistency/transactional unit) as a frozen dataclass holding an immutable collection of children, with every operation that changes membership or enforces cross-child invariants implemented as a pure function returning a new aggregate — never a mutable root class with methods that mutate a private dict in place.
---

# Python Clean Architecture: Aggregates, Functional-Lite

A DDD aggregate is a cluster of entities/value objects treated as one unit
for consistency and transactional purposes — a root entity (e.g., `Project`)
owning a bounded set of children (e.g., its `Task`s), with invariants that
span the whole cluster enforced at the aggregate boundary. Reference
material implements this as a mutable root class wrapping a private `dict`,
with methods (`add_task`, `remove_task`) mutating that dict in place. In
functional-lite, the aggregate is a frozen dataclass holding an immutable
collection of children, and every operation that changes membership becomes
a pure function returning a new aggregate — the same transition-function
pattern already used for single entities, applied at the cluster level.

---

## When to use this skill

Use this skill when you need to:

- model a root entity that owns and manages a bounded collection of other
  entities (a project and its tasks, an order and its line items),
- enforce an invariant that spans multiple child entities, not just one
  (no duplicate task titles within a project, an order total matching the
  sum of its line items),
- translate a DDD aggregate example (a root class with a private mutable
  collection and mutating methods) into this repo's style,
- decide what should count as one atomic "transaction" when several child
  entities change together,
- decide what a repository should load and save as a single unit (see
  python-clean-architecture-dependency-rule).

---

## Outcome

Produce an aggregate implementation that:

- represents the root and its children as a frozen dataclass holding an
  immutable collection (a `tuple`, or a `Mapping` built via
  `types.MappingProxyType`) — never a mutable class wrapping a private
  `dict` that callers or methods mutate in place,
- implements every membership-changing operation (`add_task`,
  `remove_task`) as a pure function taking the current aggregate and
  returning a new one via `dataclasses.replace`, exactly like a single
  entity's transition functions,
- enforces cross-child invariants (no duplicate titles, a total that must
  match a sum) inside those same functions, rejecting an operation that
  would violate the invariant rather than allowing the aggregate to reach
  an inconsistent state,
- treats "changes to several children happening together" as a single
  function call returning a single new aggregate value — the natural
  functional equivalent of a transactional boundary, since there's no
  intermediate, partially-updated state ever observable,
- exposes children only through the aggregate's own functions — external
  code reads the children via a plain accessor, but never has a route to
  add/remove/replace a child that bypasses the aggregate's invariant checks.

---

## Instructions for the AI

1. **Represent the root as a frozen dataclass holding an immutable collection**
   - Translate a mutable `Project` (with a private `dict[UUID, Task]` and
     mutating `add_task`/`remove_task`/`get_task` methods) into a frozen
     dataclass holding a `tuple[Task, ...]`:
     ```python
     from dataclasses import dataclass, field
     from uuid import UUID, uuid4

     @dataclass(frozen=True)
     class Project:
         id: UUID = field(default_factory=uuid4)
         name: str = ""
         description: str = ""
         tasks: tuple[Task, ...] = field(default_factory=tuple)
     ```
   - A tuple (not a list or dict) makes the collection itself immutable —
     there's no `.append()` or `[key] = value` available to accidentally
     mutate in place, which is what made the private-dict version need
     mutating methods to control access in the first place.

2. **Implement membership changes as pure functions returning a new aggregate**
   - Translate `add_task`/`remove_task` into functions that build and
     return a new `Project`, exactly like single-entity transition
     functions (see python-clean-architecture-entity-invariants):
     ```python
     from dataclasses import replace

     def add_task(project: Project, task: Task) -> Project:
         if any(t.title == task.title for t in project.tasks):
             raise ValueError(f"A task titled '{task.title}' already exists in this project")
         return replace(project, tasks=project.tasks + (task,))

     def remove_task(project: Project, task_id: UUID) -> Project:
         return replace(project, tasks=tuple(t for t in project.tasks if t.id != task_id))

     def get_task(project: Project, task_id: UUID) -> Task | None:
         return next((t for t in project.tasks if t.id == task_id), None)
     ```
   - Usage looks the same in shape as the mutating version, but each step
     produces a new value rather than changing the existing one:
     ```python
     project = Project(name="Website Redesign")
     project = add_task(project, task1)
     project = add_task(project, task2)

     print(f"Project: {project.name}")
     print(f"Number of tasks: {len(project.tasks)}")
     print(f"First task: {project.tasks[0].title}")
     ```

3. **Enforce cross-child invariants inside the aggregate's own functions**
   - Any rule spanning the whole collection (no duplicate titles, a
     maximum task count, a total that must reconcile) belongs inside the
     aggregate-level function that changes membership — exactly where the
     mutable version's `add_task` would have checked it before mutating,
     translated into a check-then-return-new-value shape (as shown in
     `add_task` above).
   - This is what preserves the aggregate's core promise: it should be
     structurally impossible to reach an inconsistent collection state,
     because the only way to get a new `Project` value with a different
     `tasks` collection is through a function that's already checked the
     invariant.

4. **Preserve encapsulation without needing access control**
   - The classic aggregate's "encapsulation" (external code can't directly
     modify the task collection) doesn't require a private attribute and
     getter/setter methods in functional-lite — it's structural: `tasks`
     can be read directly (`project.tasks`), but there is no mutating
     operation on a tuple to call, so the only route to a *different*
     `Project` value is through `add_task`/`remove_task`, which enforce
     the invariants. Nothing needs to be hidden because there's nothing
     mutable to protect.

5. **Treat multi-child operations as a single function call, i.e., the transactional boundary**
   - When an operation needs to change several children at once (e.g.,
     marking every task in a project complete), implement it as one
     function that builds the entire new collection and returns one new
     `Project` — never as a loop of individual mutations applied to a
     shared mutable aggregate:
     ```python
     def complete_all_tasks(project: Project) -> Project:
         return replace(project, tasks=tuple(complete_task(t) for t in project.tasks))
     ```
   - This function either succeeds and returns a fully-updated `Project`,
     or raises and returns nothing — there's no intermediate state where
     some tasks are completed and others aren't, which is exactly what a
     transactional boundary is supposed to guarantee, achieved here
     without any actual transaction machinery.

6. **Recognize identity within the aggregate as still just the child's own `id`**
   - A child entity still carries its own global `id` (a `Task`'s `UUID`
     doesn't change because it's inside a `Project`) — there's no need for
     a separate "position within the aggregate" identity concept in the
     functional-lite version; `get_task`/`remove_task` above key off the
     child's own `id` field directly. If genuine positional/ordering
     semantics matter (e.g., task order within a project), represent that
     explicitly as the tuple's order, not as a second identity system.

7. **Connect the aggregate to persistence as a single unit**
   - When designing the repository for an aggregate (see
     python-clean-architecture-dependency-rule), load and save the whole
     aggregate — root plus children — as one unit through one `Callable`
     pair (e.g., `get_project: Callable[[UUID], Project]`,
     `save_project: Callable[[Project], None]`), rather than separate
     repository functions for the root and each child type. This mirrors
     the aggregate's conceptual promise (one consistency boundary) at the
     persistence boundary as well.
   - Keep aggregates small. If a `Project` could grow to hold thousands of
     tasks, loading/saving the entire tuple on every change becomes
     expensive — treat a growing aggregate as a signal to reconsider its
     boundary (e.g., query tasks separately for read-heavy views, while
     still funneling all *writes* that affect invariants through the
     aggregate's functions) rather than accepting an ever-larger tuple by
     default.

---

## Decision points and guidance

- **Does this entity need to enforce a rule about a collection of other
  entities it owns?** That's the signal for an aggregate — the collection
  becomes an immutable field, and the rule lives in the functions that
  change membership.
- **Is a mutating method wrapping a private `dict`/`list` being
  translated?** Replace the private mutable collection with an immutable
  one (`tuple`), and the mutating methods with functions returning a new
  aggregate value.
- **Does an operation touch several children at once?** Implement it as one
  function producing one new aggregate value, not a loop of individual
  mutations — that's what preserves the transactional-boundary guarantee.
- **Is the aggregate growing large?** Treat that as a prompt to reconsider
  the boundary or split read paths from the write-invariant path, not as
  something to silently accept.

---

## Quality criteria

A strong functional-lite aggregate implementation should ensure that:

- **the root is a frozen dataclass holding an immutable collection**, never
  a mutable class wrapping a private `dict`/`list`,
- **every membership-changing operation is a pure function** returning a
  new aggregate value, with the same validation the mutating version had,
- **cross-child invariants are enforced inside those functions**, making an
  inconsistent collection state unreachable,
- **multi-child changes happen as one function call**, never as a sequence
  of individual mutations against a shared aggregate,
- **children retain their own identity** (their own `id` field), with no
  separate positional-identity system unless ordering is a genuine domain
  concern (in which case it's just the tuple's order),
- **the aggregate is the persistence unit** — one repository load/save per
  aggregate, not per child — and its size is actively watched.

---

## Example prompts

- "This `Project` aggregate wraps a private dict with mutating
  `add_task`/`remove_task` methods — reformulate it as an immutable
  collection with pure functions."
- "We need to enforce 'no duplicate task titles in a project' — where does
  that rule belong, and how do I implement it without mutation?"
- "This operation needs to mark every task in a project complete at once —
  how do I do that without an intermediate half-updated state?"

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-entity-invariants
- python-clean-architecture-domain-modeling
- python-clean-architecture-dependency-rule
