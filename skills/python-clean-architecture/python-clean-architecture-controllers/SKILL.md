---
name: python-clean-architecture-controllers
description: Instructs the agent to implement Interface Adapters controllers as plain functions receiving the use-case function and presenter function as Callable parameters — never a class with a handle_x method. Reuses the same generic Result[T] tagged union from the Application layer rather than inventing a second, differently-shaped OperationResult type. Controllers stay framework-agnostic (no FastAPI/Click imports) — that binding belongs one layer further out.
---

# Python Clean Architecture: Controllers, Functional-Lite

A controller handles inbound flow: accept external input, validate/convert
it, call a use case, hand the result to a presenter. Reference material
implements this as a class with injected dependencies and a `handle_x`
method — the same "interactor" shape already reformulated for use cases
(see python-clean-architecture-use-cases), reapplied one layer out. The
fix is identical: a controller is a function, its dependencies are
`Callable` parameters, and calling it with those dependencies supplied is
the entire "injection" step.

---

## When to use this skill

Use this skill when you need to:

- implement a controller that accepts external input and coordinates a use
  case,
- translate a controller class (`handle_create` method, constructor-
  injected use case and presenter) into this repo's style,
- decide whether a controller needs its own result type, or should reuse
  the one already established for use cases,
- review a controller for accidental framework coupling (importing a web
  framework, handling HTTP-specific concerns) that belongs one layer out.

---

## Outcome

Produce controllers that:

- are plain functions, not classes — the use case and presenter arrive as
  `Callable` parameters, exactly like any other dependency-inverted
  function in this codebase,
- return the same generic `Result[T]` (`Success[T] | Failure`) already
  established in python-domain-error-handling — never a second,
  differently-shaped result type invented per layer,
- accept only primitive types and other plain values from callers, never
  a framework-specific request object (`FastAPI`'s `Request`, a CLI
  library's context object),
- contain zero imports from a specific web/CLI framework — that binding
  happens in a thin wrapper one layer further out (`frameworks/`), never
  inside the controller itself.

---

## Instructions for the AI

1. **Reformulate a controller class as a function**
   - Translate a frozen-dataclass controller with a `handle_create`
     method:
     ```python
     @dataclass
     class TaskController:
         create_use_case: CreateTaskUseCase
         presenter: TaskPresenter

         def handle_create(self, title: str, description: str) -> OperationResult[TaskViewModel]:
             ...
     ```
     into a plain function taking the use case and presenter as `Callable`
     parameters:
     ```python
     CreateTaskUseCase = Callable[[CreateTaskRequest], Result[Project]]
     PresentTask = Callable[[Project], TaskViewModel]

     def handle_create_task(
         create_task: CreateTaskUseCase,
         present_task: PresentTask,
         title: str,
         description: str,
     ) -> Result[TaskViewModel]:
         try:
             request = CreateTaskRequest(title=title, description=description)
         except ValueError as e:
             return Failure(Error.validation_error(str(e)))

         result = create_task(request)
         match result:
             case Success(task):
                 return Success(present_task(task))
             case Failure(error):
                 return Failure(error)
     ```
   - There is no `TaskController(create_use_case=..., presenter=...).
     handle_create(...)` two-step call — `handle_create_task(create_task,
     present_task, title, description)` is the entire call, with
     dependency injection folded into the argument list. For a set of
     related handlers sharing the same dependencies, either pass the
     dependencies to each handler function individually, or bundle them in
     a `NamedTuple` per python-clean-architecture-interface-segregation if
     genuinely every handler needs the same full set.

2. **Reuse the existing generic `Result[T]` — don't invent a per-layer variant**
   - Reject introducing a second result type (`OperationResult[T]` with
     `_success`/`_error` fields) for controllers — this is the same
     two-optional-fields design already identified as a problem in
     python-domain-error-handling (an invalid state — both set, or
     neither — is representable, and every consumer has to check a
     boolean property before the type checker can narrow anything).
   - Use the same `Result[T] = Success[T] | Failure` tagged union
     established there for controller return types too — just with a
     different `T` (`Result[TaskViewModel]` instead of `Result[Project]`).
     A generic type is generic precisely so it doesn't need to be
     redefined per layer.
   - If a genuine reason exists to distinguish "a use-case-level result"
     from "a controller-level result" in a specific codebase, express that
     as a different `T` (`Result[Project]` vs. `Result[TaskViewModel]`),
     not as a structurally different result wrapper.

3. **Keep controllers framework-agnostic**
   - A controller function should never import the web framework
     (`fastapi` — the only backend framework this repo uses, see the
     `python` master skill) or a CLI framework (`click`, `typer`) directly
     — translate an anti-pattern like:
     ```python
     class WebTaskController:
         def __init__(self, app: FastAPI):
             self.app = app
         async def handle_create(self, request: Request):
             data = await request.json()
             ...
             return JSONResponse(status_code=201, content={"task": result})
     ```
     by keeping `handle_create_task` exactly as in step 1 — accepting
     plain `title: str, description: str` parameters and returning a
     `Result[TaskViewModel]` — and pushing all FastAPI-specific concerns
     (parsing `request.json()`, building a `JSONResponse`, raising
     `HTTPException`) into a separate, thin wrapper function that lives in
     `frameworks/` and calls `handle_create_task` internally:
     ```python
     # frameworks/web/routes.py — the framework binding, not the controller
     @app.post("/tasks")
     async def create_task_route(request: Request):
         data = await request.json()
         result = handle_create_task(create_task, present_task, data["title"], data["description"])
         match result:
             case Success(view_model):
                 return JSONResponse(status_code=201, content=asdict(view_model))
             case Failure(error):
                 raise HTTPException(status_code=400, detail=error.message)
     ```
   - This split means the same `handle_create_task` function can be reused
     verbatim behind a CLI, a message-queue consumer, or a different web
     framework, with only the thin outer wrapper changing per delivery
     mechanism — the concrete payoff of keeping the controller itself
     framework-agnostic.

4. **Let request models handle validation, not ad hoc controller checks**
   - Controllers construct and rely on request models (see
     python-clean-architecture-request-response-models) for validation —
     the `try/except ValueError` around request construction in step 1's
     example is catching the request dataclass's own `__post_init__`
     validation, not reimplementing validation logic inline in the
     controller.

5. **Resolve the ABC-vs-duck-typing question by not having a class at all**
   - Reference material sometimes frames a choice between a formal ABC
     interface and Python's duck typing for the use case a controller
     depends on. In this repo, that question doesn't arise: the use case
     is a function with a `Callable` type (see
     python-clean-architecture-dependency-inversion), and passing a
     differently-implemented function that matches the same `Callable`
     signature *is* the flexibility both ABC and duck typing are trying to
     provide — achieved with less machinery than either.

---

## Decision points and guidance

- **Is a controller modeled as a class with a `handle_x` method?**
  Reformulate as a function; constructor-injected dependencies become
  parameters.
- **Is a new result type being introduced for this layer?** Stop — reuse
  the existing generic `Result[T]`, just with a different type parameter.
- **Does the controller import a web/CLI framework, or handle a
  framework-specific request/response object?** Move that binding to a
  thin wrapper in `frameworks/`; the controller itself takes and returns
  plain types.
- **Is validation being reimplemented inline in the controller?** Push it
  into the request model's `__post_init__` instead, and let the controller
  just construct the request and catch `ValueError`.

---

## Quality criteria

A strong functional-lite controller implementation should ensure that:

- **every controller is a function**, with dependencies as `Callable`
  parameters, never a class with a handler method,
- **the same generic `Result[T]` is reused** across every layer that needs
  explicit success/failure, with no per-layer result type reinvented,
- **no controller imports a specific framework**, and framework-specific
  request/response handling lives in a separate outer wrapper,
- **validation lives in the request model**, not duplicated inline in the
  controller.

---

## Example prompts

- "This `TaskController` class has a `handle_create` method — reformulate
  it as a function."
- "We have both a `Result` type from the use-case layer and an
  `OperationResult` type for controllers — can we just use one?"
- "This controller imports FastAPI directly — help me separate the
  framework binding from the actual controller logic."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-presenters
- python-clean-architecture-interface-adapters-boundary
- python-clean-architecture-use-cases
- python-clean-architecture-request-response-models
- python-clean-architecture-fastapi-boundary
- python-domain-error-handling
