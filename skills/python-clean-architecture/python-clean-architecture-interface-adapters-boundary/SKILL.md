---
name: python-clean-architecture-interface-adapters-boundary
description: Instructs the agent to distinguish the Interface Adapters layer's responsibility (translating between use-case formats and external-interface formats — primitive types, framework-specific concerns) from the Application layer's (translating between domain entities and use-case formats), and to apply a concrete test for whether an adapter is needed at all — only when real format/protocol conversion is required, not whenever one layer calls into another.
---

# Python Clean Architecture: The Interface Adapters Boundary

The Interface Adapters layer and the Application layer both transform data,
which makes their boundary easy to blur. This skill covers the concrete
distinction between them, a direct test for whether an adapter is needed at
a given boundary at all (many boundaries don't need one), and applying
Interface Segregation to keep interface-adapter contracts narrow — all as
the conceptual companion to python-clean-architecture-controllers and
python-clean-architecture-presenters, which cover the concrete
implementation.

---

## When to use this skill

Use this skill when you need to:

- decide whether a piece of data-transformation code belongs in the
  Application layer or the Interface Adapters layer,
- decide whether a boundary between two layers needs an adapter at all, or
  whether the inner layer's interface can be implemented directly,
- design a set of interface-adapter `Callable` types with the right
  granularity — not one bundled interface, not needlessly many tiny ones,
- explain to a teammate why "both layers transform data" doesn't mean
  they're redundant.

---

## Outcome

Produce a layer boundary that:

- keeps Application-layer transformations working between domain entities
  and use-case-specific formats — still domain-typed, still business-rule
  focused,
- keeps Interface-Adapters-layer transformations working between use-case
  formats and external-interface formats — primitive types, framework- or
  protocol-specific shapes,
- introduces an adapter (a controller, a presenter, a `Callable`
  translation function) only where genuine format or protocol conversion
  is required — never merely because one layer is calling into another,
- defines interface-adapter `Callable` types narrowly, one per distinct
  capability, rather than one large bundled interface covering every
  operation a component might need.

---

## Instructions for the AI

1. **Distinguish the two layers' transformation responsibilities**
   - **Application layer** (see python-clean-architecture-use-cases,
     python-clean-architecture-request-response-models): transforms
     between domain entities/value objects and use-case-specific
     parameters/results. Still works in domain vocabulary — a `Project`
     entity, a `Priority` enum, a `UUID`.
   - **Interface Adapters layer** (controllers, presenters): transforms
     between use-case formats and whatever an external interface needs —
     primitive types (raw strings, dicts), or a format specific to a
     transport (HTTP JSON, CLI argument strings).
   - Use this concrete test when unsure where a transformation belongs:
     does it work with domain types on at least one side (→ Application
     layer), or does it work with pure primitives/external-protocol shapes
     on both sides (→ Interface Adapters layer)?

2. **Apply the "does this need an adapter at all" test**
   - Before introducing a controller, presenter, or any other translation
     function at a boundary, ask: does crossing this boundary require
     actual format conversion, or can the outer layer implement the inner
     layer's interface directly?
   - Example: a repository port (`GetTask = Callable[[UUID], Task]`)
     implemented directly by a database-backed function
     (`get_task_from_sqlite: GetTask`) needs no adapter — the outer
     function's signature already matches what the inner layer expects,
     with no format translation in between.
   - Contrast this with a controller or presenter, which by definition
     bridges *different* shapes (external primitive input → domain-typed
     use-case parameters; domain-typed use-case result → primitive view
     model) — that's exactly the case where an adapter earns its keep.
   - Recommend against introducing a translation function "for
     consistency" at a boundary that doesn't actually need one — this adds
     indirection without a corresponding benefit, and every unnecessary
     adapter is one more thing to keep in sync when either side changes.

3. **Design `Callable` interface-adapter types narrowly (ISP applied to this layer)**
   - Prefer several small `Callable` types over one bundled interface
     covering many operations — e.g., separate `Callable` types for task
     creation, completion, and querying, rather than one `TaskOperations`
     type with methods for all three:
     ```python
     CreateTask = Callable[[CreateTaskRequest], Result[Project]]
     CompleteTask = Callable[[UUID], Result[Project]]
     QueryTasks = Callable[[UUID], list[Task]]
     ```
   - This is the same discipline already established in
     python-clean-architecture-interface-segregation, applied specifically
     to controller/presenter/use-case boundaries: a controller handling
     task creation only needs `CreateTask` and `PresentTask` passed to it
     — not a bundle covering completion and querying it never calls.
   - When a genuine need to pass several related capabilities together
     does arise (e.g., a set of handlers sharing the exact same
     dependencies), bundle them in a `NamedTuple` at that point — not
     preemptively, and not as a single ABC-style catch-all interface.

4. **Recognize this layer's Dependency Rule direction explicitly**
   - Interface adapters depend on Application-layer `Callable` types (a
     controller's parameter is typed as `CreateTaskUseCase = Callable[...]`
     defined alongside the use case) — the Application layer never imports
     from or knows about anything in the Interface Adapters layer. This is
     the same inward-pointing dependency direction already established in
     python-clean-architecture-dependency-rule, reapplied one layer out:
     controllers and presenters are themselves an "outer" layer relative
     to use cases, even though they're "inner" relative to
     `frameworks/`.

---

## Decision points and guidance

- **Does this transformation work with domain types on at least one
  side?** Application layer. Pure primitives/external-protocol shapes on
  both sides? Interface Adapters layer.
- **Can the outer implementation match the inner layer's `Callable` type
  directly, with no format conversion?** No adapter needed — a plain
  function implementing that type is sufficient.
- **Is a bundled, many-operation `Callable`/interface type being defined
  for a controller or presenter dependency?** Split it into narrower,
  single-capability types unless there's a concrete reason several travel
  together everywhere they're used.
- **Does a controller or presenter import from `frameworks/`, or vice
  versa incorrectly?** Check the dependency direction — it should always
  point from `frameworks/` inward toward `interfaces/`/`application/`, never
  the reverse.

---

## Quality criteria

A strong Interface Adapters boundary should ensure that:

- **the Application/Interface-Adapters distinction is applied
  consistently**, using the domain-type-on-at-least-one-side test,
- **adapters exist only where format/protocol conversion is genuinely
  needed**, not at every layer crossing by default,
- **interface-adapter `Callable` types are narrow and capability-specific**,
  not bundled into large catch-all interfaces,
- **dependencies point inward** through this layer exactly as they do
  everywhere else in the architecture.

---

## Example prompts

- "Does this data transformation belong in the Application layer or the
  Interface Adapters layer?"
- "Do we actually need a controller here, or can the caller use the
  use-case function directly?"
- "We have one big interface covering task creation, completion, and
  querying — should we split it up?"

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-controllers
- python-clean-architecture-presenters
- python-clean-architecture-interface-segregation
- python-clean-architecture-dependency-rule
- python-clean-architecture-use-cases
