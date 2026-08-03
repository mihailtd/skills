---
name: python-clean-architecture-dependency-inversion
description: Instructs the agent to achieve Clean Architecture's dependency inversion (depend on abstractions, not concrete implementations) using plain functions and Callable type aliases — never ABCs, and Protocol only for typing bundles of related callables — instead of the ABC-subclass or Protocol-class patterns most Clean Architecture material demonstrates. The "abstraction" is a function signature; the "concrete implementation" is a function matching it, passed as a parameter, never a class injected into another class's constructor.
---

# Python Clean Architecture: Dependency Inversion, Functional-Lite

Clean Architecture material almost universally teaches dependency inversion
through class hierarchies — an `ABC` (or `Protocol`) defining an interface,
concrete classes implementing it, and a service class that receives an
instance via constructor injection. **This repo does not use that
mechanism.** Dependency inversion here is achieved with plain functions:
the abstraction is a function's type signature (`Callable[...]`), the
concrete implementation is a function matching that signature, and
"injection" is simply passing that function as a parameter. This skill
covers how to do that correctly, including the one narrow case where
`Protocol` is still useful — and the case (`ABC`) that's never appropriate
for business-logic dependency inversion in this repo.

---

## When to use this skill

Use this skill when you need to:

- implement a piece of business logic that needs to depend on a
  swappable behavior (e.g., "notify a customer," "look up a price,"
  "persist an entity") without hardcoding which concrete implementation
  it uses,
- translate an ABC-based or Protocol-based Clean Architecture example
  (from a book, article, or existing codebase) into this repo's style,
- decide whether a dependency should be a single passed-in function, or a
  small bundle of related functions,
- write a unit test for code that has a dependency, without reaching for a
  mocking library.

---

## Outcome

Produce dependency-inverted code that:

- expresses every "abstraction" as a `Callable[[...], ...]` type alias (or
  a `NamedTuple`/frozen `dataclass` of such callables when several related
  operations need to travel together), never as an `ABC` or a class with
  abstract methods,
- expresses every "concrete implementation" as a plain function matching
  that signature, never as a class inheriting from or structurally
  conforming to an interface class,
- "injects" a dependency by passing the function (or bundle of functions)
  as a regular parameter to whatever function needs it — no constructors,
  no DI containers, no `self.dependency = dependency`,
- can be swapped for a different implementation by passing a different
  function — the exact same flexibility ABC/Protocol-based examples
  advertise, achieved with strictly less code,
- can be tested by passing a small stand-in function directly, with no
  mocking library, no subclassing, and no `Mock()` object required unless
  one is genuinely convenient.

---

## Instructions for the AI

1. **Remember who owns the abstraction — the high-level code, not the low-level implementation**
   - DIP's actual insight isn't just "depend on abstractions instead of
     concrete implementations" — it's that the *high-level* module defines
     the abstraction, and *low-level* modules conform to it. Traditionally
     it's the other way around (low-level modules define the interface
     that high-level code is stuck adapting to).
   - Apply this concretely: the `Callable` type alias for a dependency
     (e.g., a `SaveUser = Callable[[User], None]` type) should be declared
     in or alongside the `application/`/`domain/` code that *needs* it,
     not alongside the `infrastructure/` code that implements it (see
     python-clean-architecture-dependency-rule's project layout — the
     "port" is declared in `interfaces/`, closer to the inward-facing side
     of the boundary, and *implemented* in `frameworks/`/`infrastructure/`).
   - A concrete illustration: translating the book's `UserEntity`/
     `DatabaseInterface`/`MySQLDatabase`/`PostgreSQLDatabase` example, the
     `SaveUser` callable type is defined next to the use case that needs
     saving, not next to the database code — `MySQLDatabase`'s and
     `PostgreSQLDatabase`'s equivalents (`save_user_mysql`,
     `save_user_postgres`) are just plain functions elsewhere that happen
     to match that already-declared signature. The use case never imports
     from the database module at all; only the outermost wiring code does.

2. **Express the abstraction as a `Callable` type, never as an ABC**
   - Where a book or article defines an interface as an `ABC` with one or
     more `@abstractmethod`s, translate each abstract method into a
     `Callable[[ArgTypes], ReturnType]` type alias instead.
   - Reject `ABC` outright for this purpose in this repo — it requires
     concrete implementations to be defined as classes inheriting from it,
     which reintroduces the exact class-hierarchy structure functional-lite
     avoids. There is no case where business-logic dependency inversion in
     this repo should reach for `abc.ABC`.
   - Example — the book's `Notifier` ABC becomes a type alias:
     ```python
     from typing import Callable

     # The "abstraction" — what the book expressed as an ABC
     NotifierFn = Callable[[str], None]
     ```

3. **Express concrete implementations as plain functions, never as subclasses**
   - Where a book defines `EmailNotifier(Notifier)` and `SMSNotifier(Notifier)`
     as classes implementing the interface, translate each into a plain
     function matching the `Callable` signature — no inheritance, no class
     at all:
     ```python
     def send_email_notification(message: str) -> None:
         print(f"Sending email: {message}")

     def send_sms_notification(message: str) -> None:
         print(f"Sending SMS: {message}")
     ```
   - These functions satisfy `NotifierFn` purely by having the right
     signature — Python's structural typing for callables makes this work
     without any inheritance or `Protocol` declaration needed at all.

4. **"Inject" the dependency as a plain function parameter, never via a constructor**
   - Where a book defines a service class whose `__init__` takes the
     interface instance and stores it as `self.dependency`, translate the
     whole thing into a plain function that takes the dependency as a
     parameter and calls it directly:
     ```python
     def notify(notifier: NotifierFn, message: str) -> None:
         notifier(message)

     # Usage — passing the function IS the "dependency injection"
     notify(send_email_notification, "Hello via email")
     notify(send_sms_notification, "Hello via SMS")
     ```
   - This achieves exactly what the book's `NotificationService(notifier)`
     constructor injection achieves — the calling code decides which
     concrete behavior to use, and `notify` never hardcodes it — with no
     class, no `self`, and no object to construct before calling.
   - The type hints still do real work here, exactly as the book intends:
     `notifier: NotifierFn` gives IDE support and static-analysis error
     detection identical to typing against an `ABC` or `Protocol`. Nothing
     about type safety is lost by dropping the class.

5. **Bundle related functions with a `NamedTuple` or frozen `dataclass`, not a class with methods**
   - When a dependency needs several related operations that travel
     together (e.g., a repository needing both a lookup and a save
     operation), don't reach for a service/repository *class* — bundle the
     functions themselves as fields of a `NamedTuple` or
     `@dataclass(frozen=True)`:
     ```python
     from typing import NamedTuple, Callable

     class CustomerRepo(NamedTuple):
         get_by_id: Callable[[str], Customer]
         save: Callable[[Customer], None]

     # A concrete bundle — plain functions grouped, not a class with methods
     sql_customer_repo = CustomerRepo(
         get_by_id=get_customer_by_id_sql,
         save=save_customer_sql,
     )

     def apply_credit_use_case(repo: CustomerRepo, customer_id: str, amount: float) -> dict:
         customer = repo.get_by_id(customer_id)
         result = apply_credit(customer, amount)  # pure core function
         if result.success:
             repo.save(result.customer)
         return {"status": "ok" if result.success else "error", "message": result.message}
     ```
   - This is the functional-lite equivalent of the classic "repository
     interface" pattern — same swappability (pass `sql_customer_repo` or
     `in_memory_customer_repo`), same testability, zero class hierarchy.

6. **Reserve `Protocol` for a narrow, specific case — never as the default**
   - `Protocol` has one legitimate use in this repo's dependency-inversion
     style: typing a bundle of callables *structurally*, when you want
     callers to be able to pass "anything with this shape" without
     importing a specific `NamedTuple` type (e.g., for a public library
     boundary). Even then, it's typing a *shape of data/callables*, never
     a class with actual business-logic methods attached.
   - Default to a `NamedTuple`/frozen `dataclass` of callables instead of
     `Protocol` whenever the calling code is within this codebase — it's
     simpler, requires no extra typing machinery, and is exactly as
     swappable.
   - Never use `Protocol` (or `ABC`) to type something that has real
     behavior beyond plain data or callables — if it's tempting to add a
     method with actual logic to a `Protocol`-typed thing, that logic
     belongs in a pure function elsewhere instead.

7. **Test by passing a stand-in function, not by mocking or subclassing**
   - Because a dependency is just a function (or a bundle of functions),
     testing needs no mocking library, no `Mock()`, and no test-only
     subclass — pass a small, purpose-built function or lambda directly:
     ```python
     def test_notify_calls_the_given_notifier():
         sent = []
         def fake_notifier(message: str) -> None:
             sent.append(message)

         notify(fake_notifier, "test message")

         assert sent == ["test message"]
     ```
   - This is the concrete payoff of functional dependency inversion: the
     same testability Clean Architecture promises, achieved with even less
     ceremony than the book's ABC-based example, since there's no
     `unittest.mock.Mock(spec=Notifier)` or fake-subclass boilerplate
     needed at all.

---

## Decision points and guidance

- **Does reference material define an `ABC` or `Protocol` interface?**
  Translate it to a `Callable` type alias (single operation) or a
  `NamedTuple`/frozen `dataclass` of callables (multiple related
  operations) before applying it — never copy the class-based structure
  as-is.
- **Does reference material define a service class with constructor
  injection?** Translate it to a plain function taking the dependency as a
  parameter — "injection" is just passing an argument.
- **Is `Protocol` being reached for?** Confirm it's typing a structural
  shape of callables/data for a public boundary, not standing in for a
  class with real behavior — if it's the latter, use a `NamedTuple`/
  dataclass of callables and a set of plain functions instead.
- **Is `ABC` being reached for at all, for business logic?** Stop — it's
  never appropriate here. Reserve real class inheritance for cases an
  external framework genuinely requires (see the `python` master skill's
  "reserve real OOP classes" rule).
- **How would this be tested?** If the answer involves a mocking library
  or a test-only subclass, check whether the dependency should instead be
  a plain function that a test can trivially replace with a stand-in.

---

## Quality criteria

A strong functional-lite dependency-inversion implementation should ensure
that:

- **no `ABC` appears in business logic** — abstractions are `Callable`
  type aliases or bundles of them,
- **no concrete "implementation" is a class** — concrete behaviors are
  plain functions matching the abstraction's signature,
- **"injection" is parameter-passing**, not constructor assignment or a DI
  container,
- **`Protocol` usage, if present, is confined to structural typing of
  callable/data bundles**, never used to describe something with real
  attached behavior,
- **tests pass in stand-in functions directly**, with no mocking library
  or test-only subclass required for a dependency that's just a function.

---

## Example prompts

- "This book's example defines a `Notifier` ABC with `EmailNotifier` and
  `SMSNotifier` subclasses — reformulate it using functions instead of
  classes."
- "This use case needs a repository with `get_by_id` and `save` — how do I
  express that dependency without a repository class?"
- "Should I use `Protocol` here, or is a `NamedTuple` of callables the
  better fit for this codebase?"
- "Write a test for this function's dependency without using `unittest.mock`."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-functional-core-imperative-shell
- python-clean-architecture-dependency-rule
- python-clean-architecture-scaling
- python-clean-architecture-open-closed
- python-clean-architecture-interface-segregation
- python-clean-architecture-substitutability
- python-clean-architecture-use-cases
- python-clean-architecture-drivers
- python-clean-architecture-composition-root
- python-clean-architecture-test-doubles
- code-review-clean-architecture-functional-style
- python-higher-order-functools
- python
