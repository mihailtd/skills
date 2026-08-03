---
name: python-clean-architecture-presenters
description: Instructs the agent to implement presenters as plain functions (a Callable type as the "port," concrete formatting functions as the implementations) instead of an ABC with private formatting methods on concrete subclasses — and to apply the Humble Object pattern by keeping views (CLI print functions, templates) free of all formatting logic, pushing every formatting decision into presenter functions where it stays independently testable.
---

# Python Clean Architecture: Presenters, Functional-Lite

A presenter handles outbound flow: transform a domain/use-case result into
a view model — pre-formatted, primitive-typed data a view can display
without any logic of its own (the Humble Object pattern). Reference
material implements a presenter as an ABC with `present_task`/
`present_error` abstract methods, concrete subclasses (`CliTaskPresenter`)
implementing them, and private `_format_*` methods for each formatting
decision. In functional-lite, the "interface" is a `Callable` type, each
concrete presenter is a plain function matching it, and each private
formatting method is just another function.

---

## When to use this skill

Use this skill when you need to:

- implement a presenter that transforms a use-case result into a view
  model for a specific interface (CLI, web, another client),
- translate a presenter ABC + concrete subclass into this repo's style,
- decide how much formatting logic belongs in a presenter versus a view,
- support multiple output formats (CLI vs. JSON) for the same underlying
  data without duplicating the view model or the use case.

---

## Outcome

Produce presenters that:

- are plain functions matching a `Callable` type — no `TaskPresenter(ABC)`
  base class, no concrete subclass implementing it,
- decompose formatting into small, independently-testable helper
  functions (the reformulated `_format_*` methods), composed by the main
  presenter function,
- produce view models that are frozen dataclasses of primitive,
  pre-formatted values — this part already needs no reformulation, since a
  view model is a value object,
- keep views (the code that actually prints/renders) genuinely humble —
  free of any formatting decision, only reading and displaying values the
  presenter already computed.

---

## Instructions for the AI

1. **Reformulate a presenter ABC and concrete subclass as `Callable` types and functions**
   - Translate:
     ```python
     class TaskPresenter(ABC):
         @abstractmethod
         def present_task(self, task_response: TaskResponse) -> TaskViewModel: ...
         @abstractmethod
         def present_error(self, error_msg: str, code: str | None = None) -> ErrorViewModel: ...

     class CliTaskPresenter(TaskPresenter):
         def present_task(self, task_response: TaskResponse) -> TaskViewModel:
             return TaskViewModel(..., status_display=self._format_status(task_response.status), ...)
         def _format_due_date(self, due_date: datetime | None) -> str:
             ...
     ```
     into `Callable` types for the "port," and plain functions for each
     concrete implementation:
     ```python
     PresentTask = Callable[[TaskResponse], TaskViewModel]
     PresentError = Callable[[str, str | None], ErrorViewModel]

     def cli_format_due_date(due_date: datetime | None) -> str:
         if not due_date:
             return "No due date"
         is_overdue = due_date < datetime.now(timezone.utc)
         date_str = due_date.strftime("%Y-%m-%d")
         return f"OVERDUE - Due: {date_str}" if is_overdue else f"Due: {date_str}"

     def cli_present_task(task_response: TaskResponse) -> TaskViewModel:
         return TaskViewModel(
             id=str(task_response.id),
             title=task_response.title,
             description=task_response.description,
             status_display=cli_format_status(task_response.status),
             priority_display=cli_format_priority(task_response.priority),
             due_date_display=cli_format_due_date(task_response.due_date),
             project_display=cli_format_project(task_response.project_id),
             completion_info=cli_format_completion_info(
                 task_response.completion_date, task_response.completion_notes
             ),
         )

     def cli_present_error(error_msg: str, code: str | None = None) -> ErrorViewModel:
         return ErrorViewModel(message=error_msg, code=code)
     ```
   - Each former private method (`_format_due_date`, `_format_status`, ...)
     becomes its own top-level function — this makes them independently
     unit-testable (call `cli_format_due_date(some_datetime)` directly and
     assert on the string) without needing to instantiate a presenter
     object first.

2. **Support multiple output formats as multiple functions matching the same `Callable` type**
   - A second interface (JSON for a web API, say) is just another function
     matching `PresentTask` — not another subclass:
     ```python
     def json_present_task(task_response: TaskResponse) -> TaskViewModel:
         return TaskViewModel(
             id=str(task_response.id),
             title=task_response.title,
             description=task_response.description,
             status_display=task_response.status.value,       # raw enum value, not a display string
             priority_display=str(task_response.priority.value),
             due_date_display=task_response.due_date.isoformat() if task_response.due_date else None,
             project_display=str(task_response.project_id) if task_response.project_id else None,
             completion_info=None,
         )
     ```
   - Which function to pass as `present_task` to a controller (see
     python-clean-architecture-controllers) is decided once, at the
     outermost wiring point — `cli_present_task` for a CLI entry point,
     `json_present_task` for a web route — with the controller function
     itself unaware of which one it received.

3. **Apply the Humble Object pattern: push every formatting decision into the presenter**
   - A view should do nothing but read already-formatted values off a view
     model and display them — no conditionals, no string formatting, no
     domain-type awareness:
     ```python
     def display_task(task_vm: TaskViewModel) -> None:
         print(f"{task_vm.status_display} [{task_vm.priority_display}] {task_vm.title}")
         if task_vm.due_date_display:
             print(f"Due: {task_vm.due_date_display}")
     ```
   - This view is already "humble" in the functional-lite version exactly
     as much as in the OOP version — the pattern's value (isolating hard-
     to-test display code from easily-tested formatting logic) doesn't
     depend on presenters being classes; it depends on *where* the
     formatting decisions live, which this skill's reformulation preserves
     exactly.
   - When reviewing a view function, treat any formatting decision found
     there (a conditional based on a domain enum, a date-formatting call,
     string concatenation beyond simple interpolation of already-formatted
     fields) as logic that escaped the presenter and needs to move back
     into a `cli_format_*`-style function.

4. **Keep view models as plain, primitive-typed frozen dataclasses**
   - `TaskViewModel` needs no reformulation — a view model is already a
     value object (see python-clean-architecture-domain-modeling): frozen,
     holding only primitives and pre-formatted strings, making no
     assumptions about how it'll be displayed. This is the one component
     in this chapter that was already correct as written.

5. **Calibrate presenter investment to how much formatting is actually needed**
   - If an application is a JSON API with a thin frontend doing its own
     formatting, a full presenter layer with extensive `cli_format_*`-
     style functions may be more machinery than the situation needs — a
     direct, minimal `entity_to_response` function (see
     python-clean-architecture-request-response-models) might suffice.
   - Reserve the full presenter pattern — dedicated formatting functions,
     multiple presenter functions per output format — for cases with
     genuinely complex or multiple-format display needs (a CLI plus a
     web UI plus a report generator sharing one set of use cases). This is
     the same "scale the pattern to what's needed" judgment as
     python-clean-architecture-scaling, applied specifically to presenters.

---

## Decision points and guidance

- **Is a presenter modeled as an ABC with a concrete subclass?**
  Reformulate as a `Callable` type and a plain function.
- **Is a private `_format_*` method being translated?** Make it a
  top-level function — this is what makes it independently testable.
- **Does a view contain any formatting decision** (a conditional, a date
  format call, anything beyond printing already-formatted fields)? Move it
  into a presenter function.
- **Are multiple output formats needed?** Write multiple functions
  matching the same `Callable` type, chosen at the wiring point — not
  multiple subclasses.
- **Does this application actually need a full presenter layer?** If
  formatting needs are minimal (a thin JSON API), a direct response
  function may be enough — don't build out formatting machinery
  speculatively.

---

## Quality criteria

A strong functional-lite presenter implementation should ensure that:

- **presenters are functions matching a `Callable` type**, never an ABC
  with concrete subclasses,
- **every formatting decision is a standalone, independently-testable
  function**, not a private method reachable only through an instantiated
  presenter,
- **views contain zero formatting logic** — they only read and display
  already-formatted view-model fields,
- **view models remain frozen dataclasses of primitives**, unchanged from
  the reference pattern,
- **presenter complexity is proportional to actual formatting need**, not
  built out by default for every project.

---

## Example prompts

- "This `TaskPresenter` ABC with a `CliTaskPresenter` subclass — reformulate
  it as functions."
- "This view function has a date-formatting conditional in it — that
  should be in the presenter, not the view. Fix it."
- "We need both a CLI and a JSON presenter for the same use case — show me
  how that works without subclassing."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-controllers
- python-clean-architecture-interface-adapters-boundary
- python-clean-architecture-domain-modeling
- python-clean-architecture-scaling
