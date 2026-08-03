---
name: python-clean-architecture-fastapi-boundary
description: Instructs the agent on where Pydantic/FastAPI fit against this package's layer structure — Pydantic BaseModel request/response DTOs live entirely in interfaces/frameworks as the FastAPI-specific implementation of python-clean-architecture-request-response-models (no parallel "internal" model needed, since domain/application never take a class-shaped DTO to begin with), and a FastAPI route handler is a thin wrapper matching on Result[T] and calling the existing framework-agnostic controller function. Also covers the API-first vs UI-centric distinction that raises the stakes on getting this boundary right.
---

# Python Clean Architecture: The FastAPI Boundary

FastAPI and Pydantic are this repo's approved web stack (see the `python`
master skill), which raises a real question: doesn't a `Pydantic.BaseModel`
crossing into request/response handling reintroduce the class-based,
framework-coupled patterns this package spends so much effort reformulating
away? The answer is no, provided Pydantic models stay exactly where
python-clean-architecture-request-response-models already puts request/
response DTOs — the `interfaces/`/`frameworks/` boundary, which was always
allowed to know about frameworks. This skill covers why that's true, why
the "duplicate internal model" dilemma classic Clean Architecture material
raises doesn't actually apply to this package's existing pattern, and the
concrete shape of a FastAPI route handler in this style.

---

## When to use this skill

Use this skill when you need to:

- decide whether a Pydantic `BaseModel` used for FastAPI request/response
  validation violates the Dependency Rule,
- write a FastAPI route handler that stays a thin adapter over an
  already-reformulated controller function,
- decide whether an API needs a separate "internal" request model
  alongside its Pydantic model,
- reason about why API-first systems (a public HTTP contract) need more
  careful boundary discipline than an internal UI a team fully controls.

---

## Outcome

Produce a FastAPI integration that:

- defines request/response DTOs as `pydantic.BaseModel` subclasses living
  entirely in `interfaces/` (or `frameworks/api/`) — never imported into
  `domain/` or `application/` — as the FastAPI-specific implementation of
  the request/response model pattern already established,
- never introduces a duplicate "internal" dataclass shadowing the Pydantic
  model field-for-field — domain/application functions already take plain
  parameters or an already-established domain dataclass, not a class-
  shaped request object, so there's nothing to duplicate,
- has route handlers that do nothing but: accept the Pydantic model
  (validated automatically by FastAPI), convert it to plain
  arguments/domain types via a free function, call the existing
  framework-agnostic controller function, and translate the `Result[T]`
  into an HTTP response,
- treats API contract stability (versioning, backward compatibility) as
  a controller/interfaces-layer concern, kept separate from how the
  domain model is free to evolve underneath it.

---

## Instructions for the AI

1. **Recognize that Pydantic models belong exactly where request/response models already live**
   - python-clean-architecture-request-response-models already scopes
     request/response DTOs to the boundary layer, converted to/from plain
     domain parameters via free functions (`request_to_params`,
     `entity_to_response`) before a use case is ever called. A
     `pydantic.BaseModel` used for a FastAPI endpoint is simply *this same
     DTO*, implemented with a framework that provides automatic validation
     instead of manual `__post_init__` checks:
     ```python
     from pydantic import BaseModel, Field

     class CreateTaskRequest(BaseModel):
         title: str = Field(..., min_length=1)
         description: str
         project_id: str | None = None
     ```
   - This is not a new architectural compromise — `interfaces/`/
     `frameworks/` was always the layer allowed to depend on frameworks.
     The Dependency Rule is preserved as long as `domain/` and
     `application/` never import from `pydantic` or receive a
     `BaseModel` instance directly.

2. **Don't introduce a duplicate "internal" model — there's nothing to duplicate**
   - Classic Clean Architecture material (including the source this
     package draws from) presents a dilemma: either let Pydantic penetrate
     inner layers, or maintain a parallel hand-written "internal" request
     class and a transformation between the two, accepting the
     duplication and maintenance cost.
   - That dilemma doesn't apply here, because
     python-clean-architecture-use-cases already established that use
     cases take plain parameters (or an existing domain dataclass), not a
     class-shaped request object — there was never a second model to keep
     in sync. The "transformation" is just the same free function this
     package already uses:
     ```python
     def create_task_params(request: CreateTaskRequest) -> dict:
         return {
             "title": request.title.strip(),
             "description": request.description.strip(),
             "project_id": UUID(request.project_id) if request.project_id else None,
         }
     ```
   - If reference material shows a separate `InternalCreateTaskRequest`
     dataclass mirroring the Pydantic model field-for-field purely to
     "protect" the domain layer, recognize that as unnecessary — the
     protection already exists structurally (the use-case function's
     parameter list, not a DTO class, is what `domain/`/`application/`
     actually depend on).

3. **Write the FastAPI route handler as a thin wrapper, not new logic**
   - The route handler's entire job is: accept the (already-validated)
     Pydantic model, convert it, call the existing controller function,
     and translate the result — no business logic, no validation beyond
     what Pydantic's field constraints already declared:
     ```python
     # infrastructure/api/routes.py
     from fastapi import APIRouter, HTTPException

     router = APIRouter()

     @router.post("/tasks/", response_model=TaskResponse, status_code=201)
     def create_task(task_data: CreateTaskRequest):
         result = handle_create_task(
             create_task_use_case, present_task,
             title=task_data.title,
             description=task_data.description,
             project_id=task_data.project_id,
         )
         match result:
             case Success(view_model):
                 return view_model  # FastAPI serializes via response_model
             case Failure(error):
                 raise HTTPException(status_code=400, detail=error.message)
     ```
   - `handle_create_task` here is the exact same controller function
     established in python-clean-architecture-controllers — FastAPI
     changes nothing about it. Only the outermost route function (and the
     Pydantic-specific request/response models it accepts/returns) is
     FastAPI-specific.
   - For FastAPI mechanics beyond this boundary concern — `APIRouter`
     organization, `Depends()`-based dependency injection, structured
     `HTTPException` usage, OpenAPI customization — see the `python-fastapi`
     package (python-fastapi-routing-validation,
     python-fastapi-dependency-injection,
     python-fastapi-response-error-handling); this skill only covers where
     those mechanics sit relative to Clean Architecture's layers.

4. **Treat API-first systems as raising the stakes on this boundary, not changing it**
   - When request/response models are consumed only by a UI the same team
     controls (a CLI, an internal web UI), a model change and its
     consumer can be updated together — the boundary is a convenience,
     not a hard contract.
   - When request/response models are a public API surface consumed by
     external clients, that same boundary becomes a stability contract —
     controllers now also carry responsibility for API versioning and
     backward compatibility, not just translation. Handle a breaking API
     change the same way multiple presenters are already handled (see
     python-clean-architecture-presenters): a new Pydantic request/
     response model pair and a new thin route function for the new
     version, both converting into the *same* underlying use case — the
     domain and application layers don't need to know a version bump
     happened at all.
   - This is also why the request/response conversion function (step 2)
     matters more in an API-first system: it's the one place a versioned
     API contract's quirks get absorbed before reaching stable, version-
     agnostic use-case code.

---

## Decision points and guidance

- **Is a Pydantic `BaseModel` being used for FastAPI request/response
  validation?** That's correct and expected — confirm it stays in
  `interfaces/`/`frameworks/` and is never imported by `domain/`/
  `application/`.
- **Is a separate "internal" dataclass being proposed to shadow a Pydantic
  model?** Check whether it's actually needed — if the use case already
  takes plain parameters, it isn't.
- **Does the route handler contain anything beyond conversion, a
  controller call, and a `Result` match?** If so, that logic has leaked
  out of the controller/use case and into the framework layer — move it
  back.
- **Is this system API-first (external clients) or UI-centric (an
  interface the team fully controls)?** API-first systems need explicit
  versioning handling at the boundary; UI-centric systems can evolve the
  boundary and its consumer together.

---

## Quality criteria

A strong FastAPI Clean Architecture boundary should ensure that:

- **Pydantic models exist only in `interfaces/`/`frameworks/`**, never
  imported by `domain/`/`application/`,
- **no duplicate internal model shadows the Pydantic model** — the
  existing free-function conversion pattern is reused as-is,
- **route handlers are thin**: validate (automatically, via Pydantic),
  convert, call the controller, translate the `Result` — nothing else,
- **API versioning is handled at the boundary** (new request/response
  models and route functions per version, same underlying use case), not
  by branching inside domain/application code.

---

## Example prompts

- "Does using a Pydantic `BaseModel` for this FastAPI endpoint violate our
  Dependency Rule?"
- "This route handler has a separate internal dataclass mirroring the
  Pydantic model — do we actually need that?"
- "Help me add a `v2` version of this endpoint without touching the
  underlying use case."
- "Review this FastAPI route handler — is it staying a thin wrapper, or
  has logic leaked into it?"

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-request-response-models
- python-clean-architecture-controllers
- python-clean-architecture-presenters
- python-fastapi-routing-validation
- python-fastapi-response-error-handling
- python-fastapi-dependency-injection
