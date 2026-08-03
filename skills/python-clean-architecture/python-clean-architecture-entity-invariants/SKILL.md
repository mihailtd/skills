---
name: python-clean-architecture-entity-invariants
description: Instructs the agent to encapsulate entity business rules (invariants) as pure state-transition functions that return a new immutable entity via dataclasses.replace, never as methods that mutate self — e.g. start_task(task) -> Task instead of task.start() mutating in place. Covers the entity-level vs. domain-level (cross-entity) rule distinction and where each kind of rule should live.
---

# Python Clean Architecture: Entity Invariants, Functional-Lite

DDD reference material encapsulates an entity's business rules (invariants)
as methods that mutate the entity in place — `task.start()` sets
`self.status = IN_PROGRESS`. That mutation is exactly what functional-lite
avoids. This skill covers the direct reformulation: every state-changing
rule becomes a pure function taking the current entity and returning a new
one via `dataclasses.replace`, raising on invalid transitions rather than
silently allowing them — the same pattern already established for the
Functional Core. It also covers the distinct question DDD raises of *where*
a rule belongs when it spans more than one entity.

---

## When to use this skill

Use this skill when you need to:

- implement a business rule that changes an entity's state (a status
  transition, a value update subject to validation),
- translate a DDD entity method (`task.complete()`, `order.cancel()`) into
  this repo's style,
- decide whether a rule belongs inside an entity's own module or in a
  domain service / use case, because it involves more than one entity,
- review code where an entity method mutates `self` and needs to be
  reformulated.

---

## Outcome

Produce entity business rules that:

- are implemented as plain functions taking the current (immutable) entity
  and returning a new entity reflecting the change, via
  `dataclasses.replace` — never as a method mutating `self`,
- raise on invalid transitions (e.g., starting an already-completed task)
  with a clear exception, exactly as the mutating version would, but
  without ever leaving the original entity in an inconsistent or
  half-updated state, since the original is never touched at all,
- live in the same module as the entity's dataclass definition when the
  rule only concerns that one entity, keeping entity + its own rules
  cohesive,
- are routed to a domain service function or a use-case function in
  `application/` when the rule spans multiple entities or needs data the
  entity itself doesn't hold.

---

## Instructions for the AI

1. **Reformulate mutating methods as pure transition functions**
   - Translate `task.start()` (which sets `self.status = IN_PROGRESS` in
     place) into a function that returns a new `Task`:
     ```python
     from dataclasses import replace

     def start_task(task: Task) -> Task:
         if task.status != TaskStatus.TODO:
             raise ValueError("Only tasks with 'TODO' status can be started")
         return replace(task, status=TaskStatus.IN_PROGRESS)

     def complete_task(task: Task) -> Task:
         if task.status == TaskStatus.DONE:
             raise ValueError("Task is already completed")
         return replace(task, status=TaskStatus.DONE)
     ```
   - The validation logic is identical to the mutating version — same
     conditions, same exception types and messages. The only change is the
     *mechanism*: instead of mutating and returning nothing, the function
     returns a new value and leaves the input untouched.
   - Callers now hold the new state explicitly, which is more honest about
     what happened than a method call that silently changed something the
     caller already had a reference to:
     ```python
     task = Task(title="Complete project proposal", priority=Priority.HIGH)
     task = start_task(task)
     print(task.status)  # TaskStatus.IN_PROGRESS

     task = complete_task(task)
     print(task.status)  # TaskStatus.DONE

     try:
         start_task(task)  # still raises, same message as before
     except ValueError as e:
         print(str(e))  # "Only tasks with 'TODO' status can be started"
     ```

2. **Keep pure, read-only entity queries as functions too, for consistency**
   - A query like `task.is_overdue()` doesn't mutate anything, so it's not
     *wrong* as a method on a frozen dataclass — but for consistency with
     the transition functions above (and with the rest of this repo's
     style), prefer a plain function taking the entity as a parameter:
     ```python
     def is_task_overdue(task: Task) -> bool:
         return task.due_date is not None and is_overdue(task.due_date)
     ```
   - This keeps every operation on `Task` — mutating-in-spirit or purely
     query — following the same calling convention (`operation(task, ...)`
     rather than a mix of `task.operation()` for some things and
     `operation(task)` for others), which makes the module easier to scan.

3. **Keep entity-local invariants alongside the entity's own module**
   - A rule that only needs the entity's own fields to validate (task
     status transitions, a deadline's "not in the past" check) belongs in
     the same module as the entity's dataclass definition — this keeps the
     entity and the rules that protect its validity cohesive and easy to
     find together, mirroring how the mutating version keeps them as
     methods on the same class.

4. **Route cross-entity rules to a domain service or use case, not the entity module**
   - When a rule genuinely needs information beyond one entity's own
     fields — e.g., "a user can't have more than five high-priority tasks
     at once," which needs the user's *other* tasks, not just the one
     being changed — that rule does not belong in the `Task` module at all.
   - Route it to a domain service function (if it's pure logic over
     several entities, no I/O) or a use-case function in `application/`
     (if it also needs to fetch related entities, which requires I/O — see
     python-clean-architecture-functional-core-imperative-shell for the
     Fetch → Compute → Persist shape this takes):
     ```python
     # domain service — pure, takes all needed entities as input
     def can_start_high_priority_task(existing_tasks: list[Task]) -> bool:
         high_priority_count = sum(
             1 for t in existing_tasks
             if t.priority == Priority.HIGH and t.status != TaskStatus.DONE
         )
         return high_priority_count < 5

     # use case — fetches what the rule needs, then applies it
     def start_high_priority_task(db, user_id: str, task_id: str) -> Task:
         existing_tasks = get_tasks_for_user(db, user_id)   # Fetch
         task = get_task_by_id(db, task_id)
         if not can_start_high_priority_task(existing_tasks):  # Compute
             raise ValueError("Cannot start: too many high-priority tasks in progress")
         updated = start_task(task)
         save_task(db, updated)                              # Persist
         return updated
     ```
   - Use this test to decide where a rule belongs: can it be validated
     using *only* the fields already on the entity being changed? If yes,
     it's entity-local. If it needs other entities, other records, or
     external state, it's a domain service or use-case concern.

5. **Never let an entity-local function reach outside the entity for data it needs**
   - If a transition function starts needing to query a database or reach
     into another entity to make its decision, that's a signal the rule
     has outgrown "entity-local" — move it to a domain service/use case
     rather than passing a database handle into what should be a pure
     function operating on one entity's own data.

6. **Never let an entity-local function perform a side effect, either**
   - The same boundary applies to side effects, not just data reads: a
     transition function like `complete_task` should never also send an
     email, write a log entry, or call an external service — even though
     doing so "feels" like part of completing a task. Translate a
     mutating method that both changes state and triggers a side effect
     (`self.status = DONE; self.send_completion_email()`) by keeping only
     the state transition in the entity-local function, and moving the
     side effect out entirely:
     ```python
     # Entity-local — pure, only the state transition
     def complete_task(task: Task) -> Task:
         if task.status == TaskStatus.DONE:
             raise ValueError("Task is already completed")
         return replace(task, status=TaskStatus.DONE)

     # Notifier — a Callable dependency, not a method on Task
     NotifyCompletion = Callable[[Task], None]

     # Use case — sequences the pure transition and the side effect
     def complete_task_and_notify(
         db, notify: NotifyCompletion, task_id: str
     ) -> Task:
         task = get_task_by_id(db, task_id)     # Fetch
         updated = complete_task(task)          # Compute (pure)
         save_task(db, updated)                 # Persist
         notify(updated)                        # Side effect, explicitly sequenced
         return updated
     ```
   - This keeps `complete_task` reusable and trivially testable (call it,
     assert on the returned `Task`, no email gets sent as a side effect of
     running a unit test) while still making the notification an explicit,
     visible step in the use case that orchestrates it — never an implicit
     consequence hidden inside what looks like a simple state change.

---

## Decision points and guidance

- **Is `self` being mutated in a translated example?** Reformulate as a
  function returning a new entity via `dataclasses.replace`, keeping the
  same validation logic and exceptions.
- **Does this rule need only the entity's own fields to validate?**
  Entity-local — keep the function in the entity's module. Does it need
  other entities or external state? Domain service or use case.
- **Is a query method pure and read-only?** Fine either as a method or a
  function, but default to a function for calling-convention consistency
  with the transition functions in the same module.
- **Does an entity-local function need to reach for I/O or other
  entities?** That's the signal to move it out to `application/` — don't
  smuggle a database handle into what should be a pure entity function.

---

## Quality criteria

A strong functional-lite entity-invariants implementation should ensure
that:

- **every state-changing rule is a pure function** returning a new entity
  via `dataclasses.replace`, never a method mutating `self`,
- **validation logic and exception messages are preserved exactly**
  through the reformulation — only the mechanism (mutation vs. return)
  changes,
- **entity-local rules live with the entity's own module**; cross-entity
  rules live in a domain service or use case,
- **no entity-local function performs I/O** or reaches into unrelated
  entities to make its decision,
- **calling convention is consistent** — `operation(entity, ...)` — across
  both transition functions and read-only queries.

---

## Example prompts

- "This `Task.start()` method mutates `self.status` — reformulate it as a
  pure function."
- "Where should this 'max five high-priority tasks' rule live — in the
  Task entity, or somewhere else?"
- "Review this use case and check whether any entity-local function is
  secretly doing I/O it shouldn't be."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-domain-modeling
- python-clean-architecture-aggregates
- python-clean-architecture-factories
- python-clean-architecture-use-cases
- python-clean-architecture-functional-core-imperative-shell
- python-clean-architecture-single-responsibility
- python-clean-architecture-dependency-rule
