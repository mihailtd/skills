---
name: python-clean-architecture-screaming-architecture
description: Instructs the agent to organize a Clean Architecture Python project's top-level structure around business domain and use cases, not frameworks or technical layers — a directory listing should scream "online bookstore," never "FastAPI app." Covers the concrete naming/layout test, and the underlying benefits (protecting the investment in domain logic, mock-free testability, technology swappability, and fit with Agile/CI/CD) that justify the pattern to a team.
---

# Python Clean Architecture: Screaming Architecture

A Clean Architecture codebase's structure should announce what the system
*does* — its business purpose and use cases — not what it's *built with*.
Robert C. Martin calls this a Screaming Architecture: looking at the
top-level layout should scream "online bookstore," not "FastAPI
application." This skill covers how to apply that test concretely in a
Python project, and the underlying benefits (protected domain investment,
mock-free testability, technology flexibility, Agile/CI/CD fit) worth
citing when justifying the pattern to a team.

---

## When to use this skill

Use this skill when you need to:

- lay out or review a project's top-level directory structure for a
  Clean-Architecture-based Python codebase,
- judge whether a project's structure reveals its business purpose or just
  its tech stack,
- justify Clean Architecture's cost (extra layering, more files) to a
  team or stakeholder in terms of concrete benefits, not just principle,
- explain how Clean Architecture relates to Agile/DevOps practices already
  in use,
- decide whether a new top-level folder belongs, or whether it's smuggling
  a framework concern into a place that should stay purpose-driven.

---

## Outcome

Produce a project structure that:

- names its `domain/` and `application/` modules after business concepts
  and use cases (`orders/`, `create_task.py`, `assign_reviewer.py`), never
  after frameworks or technical roles (`models.py`, `services.py`,
  `views.py` as the *only* organizing scheme),
- passes the "screaming" test: a new reader scanning the top level of the
  project should be able to state the system's business purpose before
  identifying which web framework or database it uses,
- keeps framework and driver names (`fastapi`, `sqlalchemy`, `asyncpg`)
  confined to `infrastructure/`, never leaking into how `domain/`/
  `application/` are organized or named,
- can be justified to a team using concrete, specific benefits — not just
  "it's cleaner" — when the extra layering is questioned.

---

## Instructions for the AI

1. **Apply the "scream" test to top-level structure**
   - When laying out or reviewing a project, check what the top-level
     `domain/` and `application/` folder and file names communicate to
     someone who has never seen the codebase — do they read as a
     description of the business (`invoicing/`, `catalog/`,
     `create_task.py`, `complete_task.py`), or as a description of
     technical machinery (`models/`, `services/`, `handlers/`, `utils/`)?
   - Recommend naming `application/` functions after the use case they
     represent, in the language the business/domain uses to describe that
     scenario — `assign_task_to_user`, not `update_task_assignee_field`.
   - Push back when a project's folder names reveal its framework before
     its purpose — that's the "FastAPI application" smell the principle
     warns against, and it's a legitimate review finding, not just a
     stylistic quibble.

2. **Keep framework/driver identity confined to `infrastructure/`**
   - Recommend that only `infrastructure/` (and, to a lesser extent,
     `interfaces/` for delivery-specific concerns like HTTP routing) ever
     reference a specific framework or driver by name in its module names
     or imports — `domain/` and `application/` should read the same
     whether the project is using FastAPI, PostgreSQL, or MongoDB (see the
     `python` master skill for this repo's actual stack choices).
   - Use this as a concrete, checkable review question: does any
     `domain/` or `application/` filename or import reference a specific
     framework, driver, or vendor? If so, that's a Dependency Rule
     violation in spirit even if it isn't one in strict import terms yet
     (see python-clean-architecture-dependency-rule for the stricter,
     mechanical import check).

3. **Cite the concrete benefits when justifying the pattern**
   - **Protects the investment in domain logic:** the time spent designing
     and implementing correct business rules is the actual value of the
     system — frameworks and persistence engines come and go, but a
     well-modeled `domain/`/`application/` layer survives a framework
     migration essentially untouched. Use this specifically when a team
     questions whether the extra layering is "worth it" — the payoff shows
     up precisely when an external dependency needs to change (a framework
     goes proprietary, a database needs to scale differently), not on day
     one.
   - **Mock-free testability:** because `domain/` and `application/` are
     pure functions over immutable data (see
     python-clean-architecture-functional-core-imperative-shell), business
     rules can be tested in isolation — no database, no web server, no
     mock/stub libraries required. This tends to produce *more* tests
     being written, not just easier ones, because the friction of writing
     a test drops substantially.
   - **Technology flexibility:** because `application/` doesn't depend on
     a specific framework or driver, outer-layer technology can change (a
     new database, a new web framework, adding a web UI on top of an
     existing CLI) without touching business logic — cite the concrete
     example of starting with a CLI for internal use and later adding a
     web interface, with zero changes to `domain/` or `application/`.
   - **Fits Agile and DevOps practices already in place:** the same clear
     separation that protects domain logic also supports iterative
     development (changing/extending functionality in response to
     changing requirements) and CI/CD (a more testable, modular system is
     easier to integrate and deploy continuously). It also helps scale
     development across teams, since different teams can work on
     different layers with minimal interference — the layer boundaries
     double as team-ownership boundaries.

4. **Position Clean Architecture relative to traditional layered architecture**
   - When a team is coming from a traditional layered architecture
     (presentation/business/data-access layers where the business layer
     often still depends on persistence concerns), explain the concrete
     difference: Clean Architecture enforces the Dependency Rule strictly
     enough that `domain/`/`application/` have zero knowledge of
     persistence details, not just a "soft" separation that's often
     violated in practice. This stricter enforcement is what actually
     delivers the flexibility and resilience benefits above — a
     partially-enforced separation delivers only partial benefit.

---

## Decision points and guidance

- **Does the top-level structure scream the business, or the framework?**
  If a new reader would identify the tech stack before the business
  purpose, that's a structural finding to raise, not a minor nit.
- **Is a framework or driver name leaking into `domain/`/`application/`
  naming?** If so, treat it the same as a Dependency Rule violation in
  spirit, and push the reference out to `infrastructure/`.
- **Is the team questioning whether the extra layering is worth it?** Cite
  the specific benefit that applies to their actual situation (an upcoming
  framework migration, a desire for more/faster tests, a need to scale the
  team across layers) rather than the principle in the abstract.
- **Is this project transitioning from a traditional layered architecture?**
  Name the concrete difference — strict enforcement of zero
  persistence-knowledge in the business layer — rather than treating the
  transition as purely cosmetic renaming.

---

## Quality criteria

A strong Screaming-Architecture review should confirm that:

- **top-level names describe the business**, not the tech stack, in
  `domain/` and `application/`,
- **use-case functions are named for their scenario**, not their technical
  operation,
- **framework/driver identity is confined to `infrastructure/`**, with
  `domain/`/`application/` reading identically regardless of which
  framework or driver is actually plugged in,
- **the benefits cited in justification are specific to the team's
  situation**, not a generic restatement of the principle,
- **the Dependency Rule is enforced strictly**, not just loosely
  approximated the way a traditional layered architecture often is.

---

## Example prompts

- "Does our current folder structure scream what this system does, or just
  what it's built with?"
- "Help me rename these `application/` functions to describe the use cases
  they represent, not their technical operations."
- "The team thinks Clean Architecture is overkill for this project — help
  me make the case using benefits that actually apply to us."
- "We're migrating off a traditional layered architecture — what actually
  needs to change, not just get renamed?"

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-dependency-rule
- python-clean-architecture-functional-core-imperative-shell
- python-clean-architecture-scaling
- python-architectural-fitness-functions
