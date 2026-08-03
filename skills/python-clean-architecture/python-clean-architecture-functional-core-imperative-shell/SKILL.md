---
name: python-clean-architecture-functional-core-imperative-shell
description: Instructs the agent to implement Clean/Onion Architecture in Python as Functional Core, Imperative Shell — pure functions operating on immutable dataclasses at the center, with a thin outer shell of plain functions (not repository classes or DI containers) handling all I/O and orchestration via a linear Receive → Fetch → Compute → Persist flow. This is the required Python house style for Clean/Onion Architecture — never implement the layers as OOP classes with methods.
---

# Python Clean Architecture: Functional Core, Imperative Shell

Clean Architecture (and its Onion Architecture variant) and this repo's
functional-lite style share the exact same goal: keeping pure business
logic isolated from unpredictable side effects. This skill defines the
**required** way to implement that separation in Python: as a **Functional
Core, Imperative Shell** — not as the class-based, dependency-injected
structure most Clean Architecture material (books, articles) illustrates.
When source material shows repository classes, interface abstractions, or
constructor-injected dependencies, translate it into this shape before
applying it.

---

## When to use this skill

Use this skill when you need to:

- structure a new feature or service around Clean/Onion Architecture
  principles in Python,
- review code that implements Clean Architecture and looks OOP-heavy
  (repository classes, injected interface objects, service classes) and
  needs to be reformulated,
- explain to a teammate how "pure core, impure shell" maps onto the
  traditional ring diagram, without introducing class hierarchies,
- decide where a given piece of logic belongs — core or shell — using a
  concrete, mechanical test rather than judgment calls.

---

## Outcome

Produce a Clean Architecture implementation that:

- expresses the entire domain/use-case layer as pure functions operating on
  immutable `@dataclass(frozen=True)` records — no methods, no `self`, no
  mutation, no I/O,
- expresses the outer layer (infrastructure, database access, HTTP
  handling) as plain functions that perform I/O, not as repository classes
  or service objects requiring constructor injection,
- follows a linear, easy-to-trace flow — Receive, Fetch, Compute, Persist —
  instead of deep dependency-injection graphs of mockable interfaces,
- can be tested at the core layer with zero mocks, since pure functions
  need no test doubles to verify,
- makes the core/shell boundary visually obvious: any function containing
  `async`, `await`, a database call, or an HTTP call is shell code, full
  stop — anything without those is core code and must stay that way.

---

## Instructions for the AI

1. **Map the traditional layers onto Functional Core / Imperative Shell**
   - **Pure Core** (Entities + Use Cases, in traditional Clean Architecture
     terms): deterministic functions that take immutable domain
     data in, and return new immutable domain data (or a result/outcome)
     out — given the same inputs, always the same outputs, with no side
     effects of any kind.
   - **Imperative Shell** (Frameworks, Gateways, Controllers, Infrastructure,
     in traditional terms): the outer layer that handles the "messy
     reality" of I/O — databases, HTTP requests, file systems, external
     APIs. Its job is to fetch raw data, hand it to the pure core, and
     persist or output whatever the core computed. It should contain as
     little decision-making logic as possible — its role is orchestration,
     not business rules.
   - State this mapping explicitly whenever explaining Clean Architecture
     to someone thinking in terms of the classic ring diagram — the rings
     are still real, but they collapse into exactly two practical
     categories for how the code is actually written in this repo: pure
     functions, and shell functions that call them.

2. **Write the core as pure functions over immutable dataclasses, never as classes with methods**
   - Domain models: `@dataclass(frozen=True)` records — plain data, no
     behavior attached.
   - Business rules: free functions that take domain records (and any
     other needed pure inputs) and return new domain records or an
     outcome — never a class with business-logic methods, and never a
     mutation of the input.
   - This is not a stylistic preference — it's the same functional-lite
     rule that already governs this codebase (see the `python` master
     skill's "Data over objects" section): reserve real OOP classes for
     cases the framework requires (ORM models, Pydantic settings), never
     for business logic.
   - Example — a pure core function and its immutable domain type:
     ```python
     from dataclasses import dataclass, replace

     @dataclass(frozen=True)
     class Customer:
         id: str
         balance: float
         is_active: bool

     @dataclass(frozen=True)
     class CreditResult:
         customer: Customer
         success: bool
         message: str

     def apply_credit(customer: Customer, amount: float) -> CreditResult:
         """Pure business rule. No DB, no logging, no mutation."""
         if not customer.is_active:
             return CreditResult(customer, False, "Customer is inactive")
         updated = replace(customer, balance=customer.balance + amount)
         return CreditResult(updated, True, "Credit applied")
     ```
   - Note the use of `dataclasses.replace()` to produce a new, updated
     record instead of mutating the original — this is the idiomatic
     functional-lite way to express "the customer changed" without ever
     mutating shared state.

3. **Write the shell as plain functions, never as repository/service classes**
   - Reformulate the common "Repository" pattern (a class with
     `get_by_id`/`save` methods) as plain functions that take whatever
     connection/client they need as a parameter and return or persist
     plain data — there is no repository *object* to instantiate or inject.
   - Example — shell functions doing I/O, explicitly not wrapped in a class:
     ```python
     def get_customer_by_id(db, customer_id: str) -> Customer:
         # Real implementation would query `db`; shown here as a stand-in.
         row = db.fetch_one("SELECT * FROM customers WHERE id = %s", customer_id)
         return Customer(id=row["id"], balance=row["balance"], is_active=row["is_active"])

     def save_customer(db, customer: Customer) -> None:
         db.execute(
             "UPDATE customers SET balance = %s, is_active = %s WHERE id = %s",
             customer.balance, customer.is_active, customer.id,
         )
     ```
   - If a database driver or ORM requires a class (e.g., a SQLAlchemy
     model, an `asyncpg` connection object), that's the framework
     boundary talking — use the class where the framework demands it, but
     don't wrap it in an additional hand-written repository *class* on top
     — a plain function accepting that object as a parameter is enough.

4. **Orchestrate with a linear Receive → Fetch → Compute → Persist flow**
   - Reject the classic Clean Architecture pattern of injecting mockable
     interface objects deep into use-case constructors. Instead, write one
     orchestrating function per use case that does exactly four things, in
     order:
     1. **Receive** — accept the incoming request/command (already
        parsed/validated at this point).
     2. **Fetch** — call a shell function to load whatever current state
        is needed.
     3. **Compute** — call a pure core function, passing in the fetched
        data; get back a new state or outcome. No I/O happens in this
        step.
     4. **Persist** — call a shell function to save the computed result
        (only if the outcome calls for it), and return a response.
   - Example — the orchestrator, as a plain function, no DI container, no
     class:
     ```python
     def handle_credit_request(db, customer_id: str, deposit_amount: float) -> dict:
         # Fetch
         customer = get_customer_by_id(db, customer_id)
         # Compute (pure)
         result = apply_credit(customer, deposit_amount)
         # Persist
         if result.success:
             save_customer(db, result.customer)
             return {"status": "ok", "message": result.message}
         return {"status": "error", "message": result.message}
     ```
   - Keep this orchestrating function itself free of business *rules* — it
     should only sequence calls and branch on the core's outcome, never
     compute the outcome itself.

5. **Use a mechanical test to catch layer violations**
   - Any function containing `async`, `await`, a database call, an HTTP
     call, `open()`, environment variable access, or `datetime.now()`/
     `random`/other non-deterministic calls is shell code — it must not
     live in, or be called from, the core.
   - Any function with none of the above, taking immutable data in and
     returning immutable data out, is core code — it must not import from
     infrastructure modules, frameworks, or perform any I/O.
   - Recommend applying this test directly during review: if a "pure"
     function fails it, the I/O needs to be extracted upward into the
     shell, with the now-needed data passed in as a plain argument instead.
   - Cross-reference python-architectural-fitness-functions for how to
     enforce this boundary automatically with `ast`-based tests once the
     layer structure exists.

6. **Treat test friction as an architecture diagnostic, not just a testing problem**
   - Tests are effectively a first-class client of the codebase — they
     exercise it and assert on results the same way any other caller
     would. That means the same architectural boundaries that apply to
     the main code apply to how it's tested.
   - Use excessive setup or mocking in a test as a direct signal that
     code is misplaced across the core/shell boundary, not as a testing
     inconvenience to work around with more fixtures or a heavier mocking
     library. In this repo's functional-lite style, this diagnostic is
     sharper than usual: a `domain/`/`application/` test that needs *any*
     mock at all is telling you a shell concern (I/O, a framework call,
     non-deterministic input) has leaked into what's supposed to be a
     pure function — the fix is to extract that concern into the shell
     and pass in the now-needed value as a plain argument, not to mock it
     away.
   - Recommend treating a core-layer test as correctly scoped only when
     it can be written as: construct plain input data, call the pure
     function, assert on the plain output data — no patching, no fakes,
     no fixtures beyond simple object construction.

---

## Decision points and guidance

- **Does this logic decide something, or does it fetch/save something?**
  Deciding belongs in the pure core; fetching/saving belongs in the shell —
  don't let a "just this once" I/O call creep into a core function.
- **Is this being modeled as a class with methods?** If it's business logic
  or a repository/service, stop and reformulate as a dataclass + functions
  before proceeding — this applies regardless of how the reference material
  (a book, a blog post, an existing OOP codebase) presents it.
- **Does the orchestrator contain business rules, or just sequencing?** If
  it's making decisions beyond "did the core report success," move that
  logic into the core.
- **Is a dependency being "injected" via a constructor, or passed as a
  plain function argument?** Prefer the latter — functional-lite avoids DI
  containers and injected interface objects in favor of directly passing
  what a function needs.

---

## Quality criteria

A strong Functional-Core/Imperative-Shell implementation should ensure
that:

- **the core is 100% pure:** every core function is deterministic, free of
  I/O, and operates on immutable dataclasses,
- **the shell is thin and boring:** shell functions orchestrate and perform
  I/O, but contain no business decision logic,
- **no business logic is expressed as a class with methods:** domain
  concepts are dataclasses; behavior is free functions,
- **the flow is linear and traceable:** each use case reads as
  Receive → Fetch → Compute → Persist, not a graph of injected
  dependencies,
- **the core is tested with zero mocks:** core unit tests pass plain data
  in and assert on plain data out,
- **the async/await/I/O test cleanly separates the two layers** for any
  function in the codebase.

---

## Example prompts

- "Implement this use case using Functional Core, Imperative Shell — no
  repository classes."
- "This code has a `CustomerRepository` class with `get`/`save` methods —
  reformulate it as plain functions."
- "Review this use-case handler and tell me whether any business logic has
  leaked into the imperative shell, or any I/O has leaked into the core."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-dependency-rule
- python-clean-architecture-dependency-inversion
- python-clean-architecture-screaming-architecture
- python-clean-architecture-scaling
- python-clean-architecture-domain-modeling
- python-clean-architecture-entity-invariants
- python-clean-architecture-testing-strategy
- python-clean-architecture-test-doubles
- python-architectural-fitness-functions
- python
