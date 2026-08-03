---
name: python-clean-architecture-composition-root
description: Instructs the agent to build the composition root — the single place at startup where all concrete functions are assembled and bound to their dependencies — as a factory function returning a NamedTuple of ready-to-call functions (via functools.partial or closures), not a mutable Application class with __post_init__ wiring. Configuration is Pydantic BaseSettings (already established in this repo), not an ad hoc classmethod-based Config class. Also covers the Frameworks-vs-Drivers distinction that determines how much Interface Adapters machinery a given external dependency actually needs.
---

# Python Clean Architecture: The Composition Root, Functional-Lite

Every piece guidance in this package so far has said "pass dependencies as
parameters" — the composition root is *where* that actually happens: the
one place at startup where every concrete function (a database-backed
`get_task`, a SendGrid-backed `notify_task_completed`) is created and bound
to whatever it needs, producing the fully-wired set of functions the rest
of the application calls. Reference material builds this as a mutable
`Application` container class with `__post_init__` wiring logic. In
functional-lite, the composition root is a factory *function* returning a
`NamedTuple` of already-bound functions — `functools.partial` (or a
closure) does the "wiring" that `__post_init__` used to do.

---

## When to use this skill

Use this skill when you need to:

- assemble the concrete implementations of every port (repository,
  notifier) and bind them to the use cases and controllers that need them,
- decide whether a new external dependency needs the full Interface
  Adapters stack (controller + presenter + view model) or just a port and
  an implementation,
- translate an `Application` container class with `__post_init__` wiring
  into this repo's style,
- handle application configuration (environment variables, `.env` files)
  feeding into that wiring,
- write a `main()` entry point that assembles the composition root and
  starts the application.

---

## Outcome

Produce a composition root that:

- is a factory function (not a class) returning a `NamedTuple` bundling
  every fully-wired use-case and controller function the application
  needs, built once at startup,
- binds each function to its concrete dependencies via `functools.partial`
  or a small closure, rather than assigning attributes during a class's
  `__post_init__`,
- reads configuration through Pydantic `BaseSettings` (see
  python-pydantic-configuration — already this repo's established
  pattern), never through a bespoke `Config` class with classmethods
  reading `os.getenv` ad hoc,
- decides how much Interface Adapters machinery a given external
  dependency needs using the Frameworks-vs-Drivers test, rather than
  building out a full controller/presenter stack for every integration by
  default.

---

## Instructions for the AI

1. **Use the existing Pydantic `BaseSettings` pattern for configuration**
   - Reject a bespoke `Config` class with classmethods reading
     `os.getenv(...)` one setting at a time:
     ```python
     class Config:
         @classmethod
         def get_repository_type(cls) -> RepositoryType: ...
         @classmethod
         def get_sendgrid_api_key(cls) -> str: ...
     ```
   - Use a single `BaseSettings` subclass instead, following
     python-pydantic-configuration's already-established pattern:
     ```python
     from pydantic_settings import BaseSettings

     class Settings(BaseSettings):
         repository_type: RepositoryType = RepositoryType.MEMORY
         data_directory: Path = Path("./data")
         sendgrid_api_key: str = ""
         notification_email: str = ""

         model_config = {"env_prefix": "TODO_"}
     ```
   - This gives validated, typed configuration in one place, checked by
     `ty` at every use site — a real improvement over the classmethod
     version, which only fails at the point each individual getter happens
     to be called.

2. **Build component factories as functions returning bound functions**
   - A "component factory" produces one or more ready-to-call functions
     for a given concrete dependency, using `functools.partial` to bind
     shared state (a connection, a config value) into a function that
     already matches its port's `Callable` signature:
     ```python
     from functools import partial

     def create_repositories(settings: Settings) -> tuple[GetTask, SaveTask, GetProject, SaveProject]:
         if settings.repository_type == RepositoryType.FILE:
             get_task, save_task = make_file_task_repository(settings.data_directory)
             get_project, save_project = make_file_project_repository(settings.data_directory)
         else:
             get_task, save_task = make_in_memory_task_repository()
             get_project, save_project = make_in_memory_project_repository()
         return get_task, save_task, get_project, save_project
     ```
   - See python-clean-architecture-drivers for `make_in_memory_task_repository`/
     `make_file_task_repository` themselves — closures returning bound
     `get`/`save` function pairs, the functional equivalent of a
     repository class instance.

3. **Assemble everything into a `NamedTuple`, not a class with `__post_init__`**
   - Translate an `Application` class with fields plus `__post_init__`
     wiring:
     ```python
     @dataclass
     class Application:
         task_repository: TaskRepository
         notification_service: NotificationPort
         task_presenter: TaskPresenter

         def __post_init__(self):
             self.create_task_use_case = CreateTaskUseCase(self.task_repository, ...)
             self.task_controller = TaskController(create_use_case=self.create_task_use_case, ...)
     ```
     into a factory function that binds each use case's dependencies with
     `partial`, then binds each controller's dependencies (including the
     now-bound use-case functions) the same way, returning the whole
     bundle as a `NamedTuple`:
     ```python
     class App(NamedTuple):
         handle_create_task: Callable[[str, str], Result[TaskViewModel]]
         handle_complete_task: Callable[[UUID], Result[TaskViewModel]]
         # ... every other controller-level function the app exposes

     def create_application(
         get_task: GetTask, save_task: SaveTask,
         notify_task_completed: NotifyTaskCompleted,
         present_task: PresentTask,
     ) -> App:
         create_task = partial(create_task_use_case, get_task, save_task)
         complete_task = partial(complete_task_use_case, get_task, save_task, notify_task_completed)

         return App(
             handle_create_task=partial(handle_create_task, create_task, present_task),
             handle_complete_task=partial(handle_complete_task, complete_task, present_task),
         )
     ```
   - Calling `app.handle_create_task(title, description)` now works
     exactly like calling `app.task_controller.handle_create(title,
     description)` did — same call-site ergonomics, no class or
     `__post_init__` wiring step involved, and `App` stays a plain,
     immutable `NamedTuple` of functions.

4. **Write `main()` as a thin assembly-and-run function**
   - The entry point itself needs little reformulation — it was already
     close to a plain function in the reference material — just update it
     to call the function-returning `create_application` and to run
     against the resulting `App` bundle instead of an `Application`
     instance:
     ```python
     def main() -> int:
         settings = Settings()
         get_task, save_task, get_project, save_project = create_repositories(settings)
         notify_task_completed = create_notification_service(settings)

         app = create_application(
             get_task, save_task, notify_task_completed, cli_present_task,
         )

         try:
             return run_cli(app)
         except KeyboardInterrupt:
             print("\nGoodbye!")
             return 0
         except Exception as e:
             print(f"Error: {e}", file=sys.stderr)
             return 1

     if __name__ == "__main__":
         sys.exit(main())
     ```
   - Keep the top-level `try/except` here exactly as the reference
     material does — catching `KeyboardInterrupt` for a clean shutdown and
     a bare `Exception` at this one outermost boundary is correct (see
     python-domain-error-handling's rule against bare excepts *inside*
     business logic — this is the one place, the actual program boundary,
     where it's appropriate).
   - The composition root's value is unchanged from the reference
     material: swapping `run_cli(app)` for a FastAPI-based entry point
     that calls the same `create_application` factory requires touching
     nothing else — every use case, controller, and driver stays exactly
     as it is.

5. **Apply the Frameworks-vs-Drivers test to scope integration work correctly**
   - **Frameworks** (the web framework, a CLI framework) impose their own
     control flow and need the full Interface Adapters stack — a
     controller to translate framework-specific input, a presenter to
     format output, view models to carry the formatted data. Building a
     CLI adapter (or a FastAPI route file) means writing this full chain.
   - **Drivers** (a database, an email client, a file system) don't
     impose control flow — they just provide a capability your code calls.
     A driver needs only a port (`Callable` type, defined alongside the
     use case that needs it) and one concrete implementation matching that
     signature (see python-clean-architecture-drivers) — no controller, no
     presenter, no view model.
   - Use this test directly when scoping a new integration: does this
     external dependency dictate *how* my code is structured (a
     framework), or does it just provide a capability my code calls on its
     own terms (a driver)? Building full controller/presenter machinery
     for a driver is unnecessary ceremony; skipping it for a genuine
     framework leaves framework-specific concerns leaking into business
     logic.

6. **Reformulate a framework adapter's cached state as a closure, not instance attributes**
   - A framework adapter that needs to hold onto some state across calls
     (e.g., a CLI's cached list of projects for display) translates the
     same way a driver's internal state does — a closure, not `self`:
     ```python
     def make_cli(app: App) -> Callable[[], int]:
         current_projects: list[ProjectViewModel] = []

         def display_projects() -> None:
             nonlocal current_projects
             current_projects = app.list_projects()
             ...

         def run() -> int:
             try:
                 while True:
                     display_projects()
                     handle_selection(app, current_projects)
             except KeyboardInterrupt:
                 print("\nGoodbye!")
                 return 0

         return run
     ```
   - This is the same closure-based pattern used for drivers (see
     python-clean-architecture-drivers) — `run = make_cli(app)` replaces
     `ClickCli(app).run()`, with `current_projects` captured by the
     closure instead of stored on `self`.

---

## Decision points and guidance

- **Is application wiring being modeled as a class with `__post_init__`?**
  Reformulate as a factory function returning a `NamedTuple` of bound
  functions, using `functools.partial` for the binding.
- **Is configuration a bespoke class with `os.getenv` classmethods?**
  Replace it with `pydantic_settings.BaseSettings` per
  python-pydantic-configuration.
- **Does a new external dependency need controllers/presenters, or just a
  port and an implementation?** Apply the Frameworks-vs-Drivers test
  before building out more machinery than the integration needs.
- **Does a framework adapter need to hold state across calls?** Use a
  closure, exactly as for a stateful driver.

---

## Quality criteria

A strong functional-lite composition root should ensure that:

- **the composition root is a function**, not a class with `__post_init__`
  wiring logic,
- **configuration goes through `BaseSettings`**, validated and typed, not
  ad hoc `os.getenv` calls scattered across classmethods,
- **every use case and controller is bound via `functools.partial` or a
  closure**, returned as a plain `NamedTuple` of ready-to-call functions,
- **the Frameworks-vs-Drivers test is applied** before building
  controller/presenter machinery for something that's actually just a
  driver,
- **`main()` stays a thin assembly-and-run function**, with the one
  legitimate bare `except Exception` at the actual program boundary.

---

## Example prompts

- "This `Application` class wires up use cases and controllers in
  `__post_init__` — reformulate it as a factory function."
- "This `Config` class reads environment variables via classmethods —
  replace it with our established Pydantic settings pattern."
- "Does this new email integration need a controller and presenter, or
  just a port and implementation?"
- "Help me write `main()` for this application using the reformulated
  composition root."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-drivers
- python-clean-architecture-use-cases
- python-clean-architecture-controllers
- python-clean-architecture-dependency-inversion
- python-pydantic-configuration
