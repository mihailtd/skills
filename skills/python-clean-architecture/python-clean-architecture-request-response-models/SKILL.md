---
name: python-clean-architecture-request-response-models
description: Instructs the agent to implement request/response DTOs as frozen dataclasses with __post_init__ validation, and their transformations (external format to domain params, domain entity to external format) as free functions rather than instance/class methods — keeping the same boundary-translation purpose (validate and convert inbound data, shape and protect outbound data) with this repo's consistent function-based calling convention.
---

# Python Clean Architecture: Request/Response Models, Functional-Lite

Request and response models are DTOs that translate data across the
Application layer's boundary — validating and converting inbound data into
domain-shaped parameters, and shaping domain entities into a safe, stable
outbound format. Because DTOs are inherently just data, this pattern is
already a near-perfect fit for functional-lite: the dataclasses need no
change at all. The one adjustment is calling convention — this repo prefers
free functions (`request_to_params(req)`, `project_to_response(project)`)
over instance/classmethod transformations, for consistency with every other
operation in this style.

---

## When to use this skill

Use this skill when you need to:

- define a request model that validates and converts external input before
  it reaches a use case,
- define a response model that shapes a domain entity for external
  consumption without leaking domain internals,
- translate a `to_execution_params`/`from_entity` method-based DTO example
  into this repo's style,
- decide what belongs in a response model versus what should stay purely
  internal to the domain.

---

## Outcome

Produce request/response models that:

- are frozen dataclasses, validating their own shape in `__post_init__`
  exactly as any other value object does (see
  python-clean-architecture-domain-modeling),
- convert between external format and domain parameters via a free
  function, not an instance method (`request_to_params(request)` rather
  than `request.to_execution_params()`),
- convert a domain entity into an external-facing shape via a free
  function, not a classmethod (`project_to_response(project)` rather than
  `CompleteProjectResponse.from_entity(project)`),
- keep use-case functions working purely with domain types — a use case
  never sees raw request dicts/strings, and external callers never see raw
  domain entities.

---

## Instructions for the AI

1. **Model a request as a frozen dataclass with `__post_init__` validation**
   - Translate a request DTO directly — this part needs no reformulation,
     since a request model is a value object:
     ```python
     @dataclass(frozen=True)
     class CompleteProjectRequest:
         project_id: str  # from the API; converted to UUID during translation
         completion_notes: str | None = None

         def __post_init__(self) -> None:
             if not self.project_id.strip():
                 raise ValidationError("Project ID is required")
             if self.completion_notes and len(self.completion_notes) > 1000:
                 raise ValidationError("Completion notes cannot exceed 1000 characters")
     ```

2. **Convert a request to use-case parameters with a free function**
   - Reformulate `request.to_execution_params()` as a plain function taking
     the request and returning the parameters a use case needs:
     ```python
     def request_to_params(request: CompleteProjectRequest) -> dict:
         return {
             "project_id": UUID(request.project_id),
             "completion_notes": request.completion_notes,
         }
     ```
   - This keeps validation (in `__post_init__`, at construction time) and
     format conversion (in the free function, at the point a use case is
     about to be called) as two distinct, clearly-separated steps — the
     same separation the method-based version has, just expressed as a
     function instead of a method.
   - Prefer passing the converted values as explicit keyword arguments to
     the use-case function directly, rather than unpacking a generic
     `dict` — a typed parameter list catches a missing/renamed key at
     type-check time (`ty`), where an untyped `dict` would only fail at
     runtime:
     ```python
     params = request_to_params(request)
     result = complete_project_use_case(
         get_project, save_project, save_task, notify_task_completed,
         project_id=params["project_id"],
         completion_notes=params["completion_notes"],
     )
     ```

3. **Model a response as a frozen dataclass, built by a free function**
   - Translate a response DTO's `from_entity` classmethod into a plain
     function taking the domain entity(ies) and returning the response
     value:
     ```python
     @dataclass(frozen=True)
     class CompleteProjectResponse:
         id: str
         status: str
         completion_date: str
         task_count: int
         completion_notes: str | None

     def project_to_response(project: Project) -> CompleteProjectResponse:
         return CompleteProjectResponse(
             id=str(project.id),
             status=project.status,
             completion_date=project.completed_at,
             task_count=len(project.tasks),
             completion_notes=project.completion_notes,
         )
     ```
   - If building the response needs data beyond the entity itself (e.g., a
     related lookup), pass that data in as additional function parameters
     — the same way any other function takes what it needs as arguments,
     rather than reaching for a service via `self` the way a method might.

4. **Keep use cases working purely with domain types**
   - A use case's own signature and body should never reference the
     request/response dataclasses directly for its core logic — those
     conversions happen at the call site, immediately before calling the
     use case (request → params) and immediately after it returns
     (entity → response):
     ```python
     def complete_project_endpoint(
         get_project, save_project, save_task, notify_task_completed,
         request: CompleteProjectRequest,
     ) -> Result[CompleteProjectResponse]:
         params = request_to_params(request)
         result = complete_project_use_case(
             get_project, save_project, save_task, notify_task_completed,
             project_id=params["project_id"],
             completion_notes=params["completion_notes"],
         )
         match result:
             case Success(project):
                 return Success(project_to_response(project))
             case Failure(error):
                 return Failure(error)
     ```
   - This keeps the request/response conversion layer thin and swappable
     — a new API version (`v2`) or a different client (mobile vs. web) can
     get its own request/response functions without touching
     `complete_project_use_case` at all.

5. **Use the response model to deliberately shape and protect what's exposed**
   - Include computed/aggregate fields the domain entity doesn't carry
     directly (`task_count` derived from `len(project.tasks)`) in the
     response function, not in the domain entity itself — this keeps the
     entity free of presentation-driven fields while still letting
     external consumers get convenient, ready-to-use data.
   - Omit fields the entity has but external consumers shouldn't see —
     the response function is the deliberate, single point where that
     decision is made, rather than leaving it to whatever serializes the
     entity elsewhere.

---

## Decision points and guidance

- **Is a DTO's own shape and validation being modeled?** No reformulation
  needed — a frozen dataclass with `__post_init__` is already correct.
- **Is a `to_execution_params`/`from_entity` method being translated?**
  Convert it to a free function with the same name pattern
  (`request_to_params`, `entity_to_response`) taking the DTO/entity as an
  explicit parameter.
- **Does a use case's signature reference a request or response
  dataclass directly?** Move the conversion to the call site instead, so
  the use case keeps working with plain domain types and parameters.
- **Is the response missing a computed field, or exposing one that
  shouldn't be public?** Fix it in the response-building function, not by
  changing what the domain entity itself carries.

---

## Quality criteria

A strong functional-lite request/response implementation should ensure
that:

- **request/response DTOs are frozen dataclasses** with `__post_init__`
  validation where applicable,
- **all conversions are free functions**, not instance/classmethods, named
  consistently (`x_to_params`, `entity_to_response`),
- **use-case functions never reference request/response types directly**
  in their own parameters or bodies — only plain domain types and
  primitives,
- **computed and omitted fields are handled deliberately** in the
  response-building function, not left to incidental serialization
  elsewhere.

---

## Example prompts

- "This `CompleteProjectRequest.to_execution_params()` method — reformulate
  it as a free function."
- "This `CompleteProjectResponse.from_entity()` classmethod — same thing,
  make it a function."
- "Does our use case function accidentally take a Request object directly
  instead of plain parameters? Check and fix it."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-use-cases
- python-clean-architecture-domain-modeling
- python-clean-architecture-fastapi-boundary
- python-domain-error-handling
