---
name: python-clean-architecture-drivers
description: >-
  Instructs the agent to implement drivers (repository/notification implementations at the outermost Frameworks and Drivers ring) as closures — a factory function that creates private state (a dict, a client) and returns a tuple/NamedTuple of plain functions closing over it — instead of a class implementing an ABC with methods mutating self. The closure is the functional-lite equivalent of an object instance: private state plus behavior, without a class.
---

# Python Clean Architecture: Drivers, Functional-Lite

A driver (a repository implementation, a third-party service client
wrapper) is inherently stateful — it has to hold a connection, an in-memory
store, or a configured client across calls. Reference material models this
state with `self` on a class implementing an ABC. In functional-lite, a
closure does exactly the same job: a factory function creates the private
state once, and returns plain functions that close over it — matching the
port's `Callable` type, with no class or inheritance involved.

---

## When to use this skill

Use this skill when you need to:

- implement a repository (in-memory, file-based, database-backed) that
  satisfies a port's `Callable` type,
- implement a third-party service client wrapper (an email/notification
  provider) that needs configuration or a client object across calls,
- translate an ABC-implementing driver class into this repo's style,
- decide how a driver should hold onto whatever state it genuinely needs
  (a dict, a file path, an API client) without reintroducing a class.

---

## Outcome

Produce drivers that:

- are built by a factory function that creates whatever private state the
  driver needs once, and returns the concrete functions (matching the
  port's `Callable` types) that close over that state,
- have zero inheritance — no `class InMemoryTaskRepository(TaskRepository)`
  — the returned functions satisfy the port purely by matching its
  `Callable` signature,
- keep their state genuinely private (only reachable through the closure's
  captured variables, never exposed as a public attribute other code could
  reach into and mutate directly),
- are freely swappable at the composition root (see
  python-clean-architecture-composition-root) — any function matching the
  same `Callable` type works, in-memory or file-based or database-backed.

---

## Instructions for the AI

1. **Build a stateful repository as a closure, not a class**
   - Translate an ABC-implementing repository:
     ```python
     class InMemoryTaskRepository(TaskRepository):
         def __init__(self) -> None:
             self._tasks: dict[UUID, Task] = {}
         def get(self, task_id: UUID) -> Task:
             if task := self._tasks.get(task_id):
                 return task
             raise TaskNotFoundError(task_id)
         def save(self, task: Task) -> None:
             self._tasks[task.id] = task
     ```
     into a factory function returning a pair of plain functions closing
     over a private dict:
     ```python
     def make_in_memory_task_repository() -> tuple[GetTask, SaveTask]:
         tasks: dict[UUID, Task] = {}

         def get_task(task_id: UUID) -> Task:
             if task := tasks.get(task_id):
                 return task
             raise TaskNotFoundError(task_id)

         def save_task(task: Task) -> None:
             tasks[task.id] = task

         return get_task, save_task
     ```
   - `tasks` is exactly as private here as `self._tasks` was in the class
     version — nothing outside `make_in_memory_task_repository`'s closure
     can reach it directly. The difference is mechanism, not the actual
     encapsulation guarantee: a closure's captured variables are just as
     inaccessible from outside as a "private" instance attribute (which
     Python doesn't truly enforce anyway).
   - Call it once, at the composition root: `get_task, save_task =
     make_in_memory_task_repository()`, then pass `get_task`/`save_task`
     into whatever use cases need them.

2. **Apply the same pattern to file-based or database-backed drivers**
   - The reformulation is identical regardless of what the driver actually
     talks to — only what's captured in the closure changes:
     ```python
     def make_file_task_repository(data_dir: Path) -> tuple[GetTask, SaveTask]:
         tasks_file = data_dir / "tasks.json"
         _ensure_file_exists(tasks_file)

         def get_task(task_id: UUID) -> Task:
             for task_data in _load_tasks(tasks_file):
                 if UUID(task_data["id"]) == task_id:
                     return _dict_to_task(task_data)
             raise TaskNotFoundError(task_id)

         def save_task(task: Task) -> None:
             ...  # read, update, write tasks_file

         return get_task, save_task
     ```
   - The use cases calling `get_task`/`save_task` don't know or care
     whether they came from `make_in_memory_task_repository()` or
     `make_file_task_repository(data_dir)` — both satisfy the exact same
     `GetTask`/`SaveTask` `Callable` types. This is the concrete payoff
     the reference material demonstrates with classes, achieved here with
     no shared base type at all.

3. **Build a third-party service wrapper the same way**
   - Translate a class wrapping a configured API client:
     ```python
     class SendGridNotifier(NotificationPort):
         def __init__(self) -> None:
             self.api_key = Config.get_sendgrid_api_key()
             self.notification_email = Config.get_notification_email()
             self._init_sg_client()
         def notify_task_completed(self, task: Task) -> None:
             if not (self.client and self.notification_email):
                 return
             try:
                 message = Mail(...)
                 self.client.send(message)
             except Exception:
                 pass  # log, don't disrupt business operations
     ```
     into a factory function closing over the configured client:
     ```python
     def make_sendgrid_notify_task_completed(
         api_key: str, notification_email: str
     ) -> NotifyTaskCompleted:
         client = SendGridAPIClient(api_key) if api_key else None

         def notify_task_completed(task: Task) -> None:
             if not (client and notification_email):
                 return
             try:
                 message = Mail(
                     from_email=notification_email,
                     to_emails=notification_email,
                     subject=f"Task Completed: {task.title}",
                     plain_text_content=f"Task '{task.title}' has been completed.",
                 )
                 client.send(message)
             except Exception:
                 pass  # log, don't disrupt business operations
         return notify_task_completed
     ```
   - Configuration values (`api_key`, `notification_email`) are passed in
     as plain parameters to the factory function — sourced from the
     `Settings` object at the composition root (see
     python-clean-architecture-composition-root), not read internally via
     `Config.get_sendgrid_api_key()`-style calls scattered inside the
     driver itself. This keeps the driver a pure function of its inputs
     for construction purposes, and keeps all configuration reading
     centralized in one place.

4. **Bundle a repository's functions with a `NamedTuple` when several call sites need the full set**
   - If many places need all of a repository's operations together (not
     just `get`/`save` individually), bundle the closure's returned
     functions into a `NamedTuple` per
     python-clean-architecture-interface-segregation, still built by the
     same factory function:
     ```python
     class TaskRepo(NamedTuple):
         get: GetTask
         save: SaveTask
         delete: Callable[[UUID], None]

     def make_in_memory_task_repository() -> TaskRepo:
         tasks: dict[UUID, Task] = {}
         return TaskRepo(
             get=lambda task_id: tasks[task_id] if task_id in tasks else _raise_not_found(task_id),
             save=lambda task: tasks.__setitem__(task.id, task),
             delete=lambda task_id: tasks.pop(task_id, None),
         )
     ```
   - Default to returning a plain tuple of individually-named functions
     (as in steps 1–3) when call sites only ever need one or two of them;
     reach for the `NamedTuple` bundle only when the full set travels
     together consistently.

---

## Decision points and guidance

- **Is a driver being modeled as a class implementing an ABC?**
  Reformulate as a factory function returning closures matching the port's
  `Callable` type(s) — no base class needed.
- **Does the driver need private state across calls (a dict, a client, a
  file path)?** Capture it as a closure variable in the factory function,
  exactly as a class would capture it as `self.something`.
- **Is configuration being read inside the driver via classmethod/env-var
  calls?** Pass configuration values in as factory-function parameters
  instead, sourced once from `Settings` at the composition root.
- **Do call sites need one or two operations, or the whole set together?**
  Individual named functions for the former; a `NamedTuple` bundle for the
  latter.

---

## Quality criteria

A strong functional-lite driver implementation should ensure that:

- **every driver is built by a factory function returning closures**, with
  no class implementing an ABC anywhere,
- **private state is captured via closure**, exactly as private as a
  class's instance attributes would be,
- **drivers are freely swappable** — any function matching the same
  `Callable` type works, regardless of what's captured inside it,
- **configuration is passed in, not read internally**, keeping the driver
  a function of its explicit inputs,
- **bundling is deliberate** — a `NamedTuple` only when the full set of
  operations genuinely travels together.

---

## Example prompts

- "This `InMemoryTaskRepository` class implements an ABC — reformulate it
  as a closure-based factory function."
- "This `SendGridNotifier` class reads its API key via `Config` internally
  — reformulate it to take configuration as parameters instead."
- "We need both an in-memory and a file-based task repository for
  testing vs. production — show me how they stay swappable without a
  shared base class."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-composition-root
- python-clean-architecture-dependency-inversion
- python-clean-architecture-interface-segregation
- python-pydantic-configuration
- python-sqlalchemy-repository (PostgreSQL driver mechanics — SQLAlchemy 2.0 models, async sessions, query optimization — wrapped in this closure pattern)
- python-beanie-documents (MongoDB driver mechanics, same wrapping pattern)
