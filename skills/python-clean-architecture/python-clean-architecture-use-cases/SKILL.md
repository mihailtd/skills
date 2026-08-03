---
name: python-clean-architecture-use-cases
description: Instructs the agent to implement Application-layer use cases as plain functions returning a Result, with dependencies (repositories, notification ports) as Callable/NamedTuple parameters — never "interactor" classes with constructor-injected dependencies and an execute method. Covers multi-step orchestration, ports for external services, adapters for third-party services (also just functions), and optional/pluggable dependencies passed as parameters instead of a mutable registered-services dict.
---

# Python Clean Architecture: Use Cases, Functional-Lite

Clean Architecture's Application layer coordinates domain objects and
external services to accomplish a use case. Reference material implements
each use case as a frozen-dataclass "interactor" — dependencies as fields,
a single `execute` method — which is a class wearing a function's clothing.
In functional-lite, a use case is a plain function: dependencies arrive as
`Callable`/`NamedTuple` parameters (see
python-clean-architecture-dependency-inversion), and calling the function
with its dependencies *is* the dependency injection. This skill covers that
reformulation for single-step and multi-step use cases, ports for external
services, adapting third-party services, and optional dependencies.

---

## When to use this skill

Use this skill when you need to:

- implement an Application-layer use case that orchestrates one or more
  domain entities and external services,
- translate a "use case interactor" class (constructor-injected
  dependencies, one `execute` method) into this repo's style,
- coordinate a multi-step operation (completing a project by first
  completing all its tasks) while keeping each step's failure handling
  clear,
- define a port for an external service (notifications, storage) and adapt
  a third-party service to it,
- make a dependency optional or pluggable without introducing mutable
  runtime "service registration."

---

## Outcome

Produce Application-layer use cases that:

- are plain functions, not classes — dependencies are `Callable`/
  `NamedTuple` parameters, and the function itself is what gets called with
  those dependencies supplied, with no object to construct first,
- return a `Result[T]` (see python-domain-error-handling) making success
  and failure explicit in the function's own signature,
- orchestrate multi-step operations as a sequence of calls to entity
  transition functions and injected I/O callables, with each domain error
  caught and translated at the point it can occur,
- define ports as `Callable` types (or `NamedTuple` bundles for multi-
  operation services), never as ABCs — and adapt third-party services to a
  port with a plain wrapper function, never an adapter class,
- accept optional dependencies as `Callable | None` parameters (or a
  `dict[str, Callable]` passed once at call time), never as a mutable
  "registered services" collection attached to the use case itself.

---

## Instructions for the AI

1. **Reformulate a use-case interactor class as a function**
   - Translate a frozen-dataclass interactor with an `execute` method:
     ```python
     @dataclass(frozen=True)
     class CompleteTaskUseCase:
         task_repository: TaskRepository

         def execute(self, task_id: UUID, completion_notes: str | None = None) -> Result:
             ...
     ```
     into a plain function taking the same dependency as a parameter:
     ```python
     GetTask = Callable[[UUID], Task]
     SaveTask = Callable[[Task], None]

     def complete_task_use_case(
         get_task: GetTask,
         save_task: SaveTask,
         task_id: UUID,
         completion_notes: str | None = None,
     ) -> Result[dict]:
         try:
             task = get_task(task_id)                              # Fetch
             completed = complete_task(task, notes=completion_notes)  # Compute (pure)
             save_task(completed)                                   # Persist
             return Success({
                 "id": str(completed.id),
                 "status": "completed",
                 "completion_date": completed.completed_at.isoformat(),
             })
         except TaskNotFoundError:
             return Failure(Error.not_found("Task", str(task_id)))
         except ValidationError as e:
             return Failure(Error.validation_error(str(e)))
     ```
   - There's no `CompleteTaskUseCase(task_repository=...).execute(...)`
     two-step call — just `complete_task_use_case(get_task, save_task,
     task_id)`. The "injection" the class's `__init__` used to do is just
     the function call's argument list.

2. **Orchestrate multi-step use cases as a sequence of calls, not class methods**
   - A use case coordinating several entities (completing a project by
     completing all its outstanding tasks, then the project itself)
     becomes a function that calls entity transition functions and
     injected I/O callables in sequence, catching domain errors once at
     the boundary:
     ```python
     GetProject = Callable[[UUID], Project]
     SaveProject = Callable[[Project], None]
     SaveTask = Callable[[Task], None]
     NotifyTaskCompleted = Callable[[Task], None]

     def complete_project_use_case(
         get_project: GetProject,
         save_project: SaveProject,
         save_task: SaveTask,
         notify_task_completed: NotifyTaskCompleted,
         project_id: UUID,
         completion_notes: str | None = None,
     ) -> Result[dict]:
         try:
             project = get_project(project_id)                     # Fetch

             for task in incomplete_tasks(project):                 # Compute, per task
                 completed_task = complete_task(task)
                 save_task(completed_task)                          # Persist, per task
                 notify_task_completed(completed_task)               # Side effect, per task

             completed_project = mark_project_completed(project, notes=completion_notes)
             save_project(completed_project)                        # Persist

             return Success({
                 "id": str(completed_project.id),
                 "status": completed_project.status,
                 "completion_date": completed_project.completed_at,
                 "task_count": len(completed_project.tasks),
             })
         except ProjectNotFoundError:
             return Failure(Error.not_found("Project", str(project_id)))
         except ValidationError as e:
             return Failure(Error.validation_error(str(e)))
     ```
   - Note `incomplete_tasks(project)` and `complete_task(task)` are the
     pure entity/aggregate functions already established (see
     python-clean-architecture-entity-invariants and
     python-clean-architecture-aggregates) — the use case's job here is
     purely sequencing: call the pure logic, call the injected I/O, in the
     right order, with errors caught once at the boundary rather than
     scattered through per-step `try` blocks.
   - If the operation needs true transactional rollback (partial task
     saves need undoing if the project save fails), keep that concern in
     the injected persistence functions or the outer transaction boundary
     (e.g., a single `save_project_and_tasks` callable that wraps a DB
     transaction) — don't build rollback logic into the use-case function
     itself, which should stay focused on sequencing, not transaction
     management.

3. **Define ports as `Callable` types, not ABCs — reinforcing dependency inversion**
   - A "port" in this chapter's terminology is exactly the `Callable`
     abstraction already established: `TaskRepository`'s `get`/`save`
     become `GetTask`/`SaveTask` callables (or one `TaskRepo` `NamedTuple`
     bundling them — see python-clean-architecture-interface-segregation
     for when to bundle vs. pass individually); `NotificationPort`'s
     `notify_task_completed` becomes a single `NotifyTaskCompleted`
     callable. There is no `class TaskRepository(ABC)` anywhere in this
     repo's version.
   - When a "port" reference example has several loosely related
     operations (`notify_task_assigned`, `notify_task_completed`, ...),
     check whether they're really needed together at every call site — if
     not, prefer several independent `Callable` types over one bundled
     interface, per python-clean-architecture-interface-segregation.

4. **Adapt a third-party service with a plain function, not an adapter class**
   - Translate an adapter class wrapping a third-party service:
     ```python
     class ModernNotificationAdapter(NotificationPort):
         def __init__(self, modern_service: ModernNotificationService):
             self._service = modern_service
         def notify_task_completed(self, task: Task) -> None:
             self._service.send_notification({"type": "TASK_COMPLETED", "taskId": str(task.id)})
     ```
     into a function matching the port's `Callable` signature directly,
     closing over whatever client it needs:
     ```python
     def make_modern_notify_task_completed(client: ModernNotificationService) -> NotifyTaskCompleted:
         def notify_task_completed(task: Task) -> None:
             client.send_notification({"type": "TASK_COMPLETED", "taskId": str(task.id)})
         return notify_task_completed
     ```
   - `make_modern_notify_task_completed(client)` returns a plain function
     matching `NotifyTaskCompleted` — pass its result directly as the
     `notify_task_completed` argument to a use case. No adapter class, no
     `NotificationPort` base type to inherit from.

5. **Pass optional dependencies as parameters, never as mutable registered state**
   - Reject the "optional services" pattern where a use-case object
     exposes a `register_service(name, service)` method that mutates an
     internal dict — note this pattern is doubly broken even on its own
     terms: declaring the use case `@dataclass(frozen=True)` and then
     mutating a dict stored in one of its fields doesn't actually achieve
     immutability, since `frozen=True` only blocks *reassigning* a field,
     not mutating a mutable object already stored in it.
   - Instead, accept optional dependencies as ordinary parameters — either
     individually as `Callable | None`, or as a plain `dict[str,
     Callable]` passed once at call time if there are genuinely many
     optional integrations:
     ```python
     def complete_task_with_extras_use_case(
         get_task: GetTask,
         save_task: SaveTask,
         notify_task_completed: NotifyTaskCompleted,
         task_id: UUID,
         track_analytics: Callable[[UUID], None] | None = None,
         log_audit: Callable[[UUID], None] | None = None,
     ) -> Result[dict]:
         task = get_task(task_id)
         completed = complete_task(task)
         save_task(completed)
         notify_task_completed(completed)

         if track_analytics is not None:
             track_analytics(completed.id)
         if log_audit is not None:
             log_audit(completed.id)

         return Success({"id": str(completed.id), "status": "completed"})
     ```
   - Whatever composed which concrete functions to pass in (including
     whether the optional ones are present at all) happens once, at the
     outermost wiring point of the application (see
     python-clean-architecture-dependency-rule's `frameworks/` layer) —
     never via a stateful `register_service` call made against an
     already-in-use use case.

---

## Decision points and guidance

- **Is a "use case" being modeled as a class with an `execute` method?**
  Reformulate as a function; the constructor's dependencies become the
  function's parameters.
- **Does the use case touch more than one entity?** Sequence pure entity/
  aggregate function calls and injected I/O calls in one function, with
  errors caught once at the boundary — not scattered per-step handling.
- **Is a "port" defined as an ABC?** Reformulate as one or more `Callable`
  types (or a `NamedTuple` bundle only when several operations genuinely
  travel together).
- **Is an adapter wrapping a third-party client in a class?** Reformulate
  as a function (or a function-returning function, if the client needs to
  be closed over) matching the port's `Callable` signature.
- **Is an "optional services" pattern mutating a dict via a
  `register_service` method?** Replace it with `Callable | None`
  parameters or a `dict[str, Callable]` passed once at call time — no
  runtime registration.

---

## Quality criteria

A strong functional-lite use-case implementation should ensure that:

- **every use case is a function**, never a class with an `execute`
  method — calling it with its dependencies is the entire "injection" step,
- **it returns a `Result[T]`** (see python-domain-error-handling), with
  domain exceptions caught and translated once at the boundary,
- **multi-step orchestration is a clear sequence** of pure entity/
  aggregate calls and injected I/O calls, not hidden inside class methods,
- **ports are `Callable` types**, not ABCs, and adapters are functions, not
  classes implementing an ABC,
- **optional dependencies are parameters**, never a mutable
  "registered services" collection with a stateful registration method.

---

## Example prompts

- "This `CompleteTaskUseCase` class has one `execute` method — reformulate
  it as a plain function."
- "This use case completes a project by completing all its tasks first —
  help me sequence that as pure functions plus injected I/O."
- "This `NotificationPort` is an ABC with a `ModernNotificationAdapter`
  implementing it — reformulate both as functions."
- "This use case has a `register_service` method for optional integrations
  — that's mutating state on a supposedly frozen object. Fix it."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-dependency-inversion
- python-clean-architecture-entity-invariants
- python-clean-architecture-aggregates
- python-clean-architecture-interface-segregation
- python-clean-architecture-request-response-models
- python-clean-architecture-controllers
- python-clean-architecture-presenters
- python-clean-architecture-interface-adapters-boundary
- python-clean-architecture-composition-root
- python-clean-architecture-drivers
- python-domain-error-handling
