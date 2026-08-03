---
name: python-clean-architecture-factories
description: Instructs the agent that Python dataclasses already replace most of the classic OOP factory pattern (auto-generated __init__, __post_init__ validation, classmethod/free-function alternative constructors) — and that the remaining case (construction needing external dependencies or cross-entity data) is a use-case function receiving those dependencies as plain Callable parameters, never a factory class with an __init__ storing injected collaborators.
---

# Python Clean Architecture: Factories, Functional-Lite

The classic factory pattern encapsulates object-construction logic —
useful in languages/styles where constructors can't easily be extended.
Python's dataclasses already cover most of that need directly
(auto-generated `__init__`, `__post_init__` validation, alternative
constructors). The one case reference material still reaches for a real
factory — construction that needs external dependencies (a repository, a
service) — has a functional-lite answer already established elsewhere in
this package: it's a use-case function receiving those dependencies as
parameters, not a factory class with an `__init__` storing collaborators.

---

## When to use this skill

Use this skill when you need to:

- decide whether a piece of construction logic needs a factory at all, or
  whether a dataclass's own features already cover it,
- add validation or an alternative named-construction path to an entity or
  value object,
- translate a factory-class example (constructor-injected dependencies,
  a `create_x` method) into this repo's style,
- implement construction that depends on other entities or external state
  (e.g., "create a task within a project, considering the assignee's
  role").

---

## Outcome

Produce construction logic that:

- relies on the dataclass's own `__init__` (auto-generated) and
  `__post_init__` for the large majority of construction needs — no
  factory class involved at all,
- uses a `@classmethod` or a plain module-level function for a named,
  alternative construction path (`create_urgent_task`), never a separate
  factory class, for cases with no external dependencies,
- implements construction that genuinely needs external dependencies
  (other entities, repositories, services) as a use-case function
  receiving those dependencies as plain `Callable` parameters — the same
  pattern already used for every other use case in this package, applied
  to a "create" operation instead of an "update" one,
- never introduces a factory class with an `__init__` storing injected
  collaborators as `self` attributes.

---

## Instructions for the AI

1. **Recognize that the dataclass's own `__init__` already replaces most factory need**
   - A dataclass's auto-generated `__init__` already handles default
     values, optional parameters, and (with type checking via `ty`) type
     consistency — this is exactly what a basic factory would otherwise
     exist to provide:
     ```python
     @dataclass(frozen=True)
     class Task:
         id: UUID = field(default_factory=uuid4)
         title: str = ""
         description: str = ""
         due_date: Deadline | None = None
         priority: Priority = Priority.MEDIUM
         status: TaskStatus = TaskStatus.TODO
     ```
   - No factory is needed to construct a `Task` in the common case — this
     is already the simplest possible way to do it in Python.

2. **Use `__post_init__` for validation, not a factory**
   - Construction-time validation belongs in `__post_init__`, exactly as
     already established for value objects (see
     python-clean-architecture-domain-modeling) — it runs once, during
     construction, and rejects invalid states before they can exist:
     ```python
     def __post_init__(self) -> None:
         if not self.title.strip():
             raise ValueError("Task title cannot be empty")
         if len(self.description) > 500:
             raise ValueError("Task description cannot exceed 500 characters")
     ```
   - This covers the "factory validates before constructing" use case
     directly, without a separate factory object.

3. **Use a classmethod or free function for alternative named constructors**
   - For a named, pre-configured construction path with no external
     dependencies, either a `@classmethod` on the dataclass or a plain
     module-level function is acceptable — both are pure, take plain
     values in, and return a new instance, with no mutation or hidden
     state involved:
     ```python
     @dataclass(frozen=True)
     class Task:
         ...

         @classmethod
         def create_urgent(cls, title: str, description: str, due_date: Deadline) -> "Task":
             return cls(title=title, description=description, due_date=due_date, priority=Priority.HIGH)
     ```
     or, as a free function, which is the more consistent choice with the
     rest of this repo's calling conventions:
     ```python
     def create_urgent_task(title: str, description: str, due_date: Deadline) -> Task:
         return Task(title=title, description=description, due_date=due_date, priority=Priority.HIGH)
     ```
   - Prefer the free-function form for new code, for consistency with
     every other operation in this style (`operation(args) -> result`,
     never a mix of `.method()` and `function()` calling conventions
     across the codebase) — but don't treat an existing `@classmethod`
     alternative constructor as something that needs urgent reformulation;
     it carries no mutation or hidden dependency risk either way.

4. **Reformulate a dependency-requiring factory as a use-case function**
   - When construction genuinely needs external collaborators — other
     entities, a repository lookup, a service call — that's the one case
     classic material still reaches for a real factory class:
     ```python
     class TaskFactory:
         def __init__(self, user_service, project_repository):
             self.user_service = user_service
             self.project_repository = project_repository

         def create_task_in_project(self, title, description, project_id, assignee_id):
             project = self.project_repository.get_by_id(project_id)
             assignee = self.user_service.get_user(assignee_id)
             task = Task(title, description)
             task.project = project
             task.assignee = assignee
             if project.is_high_priority() and assignee.is_manager():
                 task.priority = Priority.HIGH
             project.add_task(task)
             return task
     ```
   - Translate this directly into a use-case function receiving the same
     collaborators as plain `Callable` parameters instead of
     constructor-injected `self` attributes (see
     python-clean-architecture-dependency-inversion for the general
     pattern) — this is construction with dependencies, not update with
     dependencies, but the shape is identical:
     ```python
     GetProject = Callable[[UUID], Project]
     GetUser = Callable[[UUID], User]

     def create_task_in_project(
         get_project: GetProject,
         get_user: GetUser,
         title: str,
         description: str,
         project_id: UUID,
         assignee_id: UUID,
     ) -> tuple[Task, Project]:
         project = get_project(project_id)
         assignee = get_user(assignee_id)
         priority = (
             Priority.HIGH
             if project.is_high_priority and assignee.is_manager
             else Priority.MEDIUM
         )
         task = Task(title=title, description=description, priority=priority)
         updated_project = add_task(project, task)  # aggregate function, see python-clean-architecture-aggregates
         return task, updated_project
     ```
   - Note the additional fix beyond dependency style: the original
     `task.project = project` and `task.assignee = assignee` lines mutate
     a supposedly-frozen entity after construction and reach for an
     aggregate's internal collection directly (`project.add_task(task)`
     called as a mutating method) — both are addressed by combining this
     skill with python-clean-architecture-aggregates and
     python-clean-architecture-entity-invariants: the task is fully
     constructed in one step, and joining it to the project goes through
     the aggregate's own `add_task` function, returning a new `Project`
     rather than mutating the existing one in place.
   - This function is, functionally speaking, a use case — it belongs in
     `application/`, following the same Fetch → Compute → Persist shape as
     any other use case, with "compute" here meaning "construct the new
     `Task` and the updated `Project`."

---

## Decision points and guidance

- **Does this construction need anything beyond the entity's own fields?**
  If no, a dataclass's `__init__`/`__post_init__` is enough — no factory
  needed at all.
- **Is this a named, pre-configured construction path with no external
  dependencies?** A `@classmethod` or free function — prefer the free
  function for new code.
- **Does construction need a repository, a service, or another entity
  looked up externally?** That's a use-case function in `application/`,
  taking those dependencies as `Callable` parameters — not a factory class
  with an `__init__`.
- **Is a factory class storing dependencies as `self` attributes in
  reference material?** Reformulate it as a function taking those same
  dependencies as parameters.

---

## Quality criteria

A strong functional-lite approach to construction should ensure that:

- **no factory class exists for construction that a dataclass's own
  features already cover**,
- **validation lives in `__post_init__`**, not a separate factory method,
- **alternative constructors are classmethods or free functions**, with
  free functions preferred for new code,
- **dependency-requiring construction is a use-case function**, receiving
  its collaborators as parameters, living in `application/`,
- **no entity is mutated post-construction** to attach relationships —
  everything needed is either passed into the initial construction or
  handled by a separate aggregate function.

---

## Example prompts

- "Do we need a factory for this, or does a dataclass with `__post_init__`
  already cover it?"
- "This `TaskFactory` class takes a user service and project repository in
  its constructor — reformulate it as a use-case function."
- "This factory mutates the task after constructing it to attach a project
  and assignee — fix that too while reformulating."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-domain-modeling
- python-clean-architecture-dependency-inversion
- python-clean-architecture-aggregates
- python-clean-architecture-entity-invariants
