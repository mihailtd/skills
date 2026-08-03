---
name: python-clean-architecture-open-closed
description: Instructs the agent to achieve the Open-Closed Principle (extend without modifying existing code) using functools.singledispatch over immutable dataclasses instead of abstract base classes and subclass polymorphism — new "shapes"/variants register a new implementation without touching the original dispatcher function. Includes the honest expression-problem tradeoff against structural pattern matching, which is simpler for a genuinely closed set of variants but requires touching the central function to add a new one.
---

# Python Clean Architecture: Open-Closed, Functional-Lite

The Open-Closed Principle (OCP) says software should be open for extension
but closed for modification — you should be able to add new behavior
without editing existing, tested code. The classic mechanism is subclass
polymorphism (an abstract base class, concrete subclasses, a caller that
only knows the abstract type). In functional-lite, the equivalent mechanism
is `functools.singledispatch` over plain dataclasses: new variants register
a new implementation without modifying the original dispatch function. This
skill covers that reformulation, and is honest about the one case
(a genuinely closed, stable set of variants) where structural pattern
matching is simpler despite not technically satisfying OCP.

---

## When to use this skill

Use this skill when you need to:

- design a function that needs to handle a growing set of data variants
  (shapes, payment types, notification kinds) without editing itself every
  time a new variant appears,
- translate a class-hierarchy-based OCP example (abstract base class +
  subclasses) into this repo's style,
- decide between `functools.singledispatch` and structural pattern
  matching (`match`/`case`) for a specific piece of type-varying logic,
- review code that uses an `isinstance()` chain or a growing `if/elif`
  ladder to handle different data shapes and needs an extensible
  alternative.

---

## Outcome

Produce extensible, functional-lite code that:

- represents each variant as its own immutable `@dataclass(frozen=True)`,
  never as a subclass of a shared base class,
- implements the operation that varies per type as a
  `functools.singledispatch` function, with each variant's behavior
  registered via `@operation.register`, never as a method on the
  variant's class,
- allows a new variant to be added by writing a new dataclass and a new
  `@operation.register` implementation, with zero changes to the original
  dispatcher function or to any existing variant's code,
- uses structural pattern matching instead, deliberately and consciously,
  only when the set of variants is genuinely closed and unlikely to grow —
  with the tradeoff named explicitly, not defaulted into by habit.

---

## Instructions for the AI

1. **Represent each variant as a dataclass, never a subclass**
   - Translate the book's `Shape` ABC with `Rectangle`/`Circle`/`Triangle`
     subclasses into independent, unrelated dataclasses — there is no
     shared base type at all:
     ```python
     from dataclasses import dataclass

     @dataclass(frozen=True)
     class Rectangle:
         width: float
         height: float

     @dataclass(frozen=True)
     class Circle:
         radius: float
     ```
   - Note there's no `Shape` base class here — functional-lite doesn't
     need a common supertype for `singledispatch` to work; it dispatches
     on the concrete runtime type of the first argument.

2. **Implement the varying operation as a `singledispatch` function**
   - Define the operation once, with a default implementation (or one that
     raises for unhandled types), then register a specific implementation
     per variant — each registration is a plain function, not a method:
     ```python
     from functools import singledispatch
     import math

     @singledispatch
     def area(shape) -> float:
         raise NotImplementedError(f"No area implementation for {type(shape)}")

     @area.register
     def _(shape: Rectangle) -> float:
         return shape.width * shape.height

     @area.register
     def _(shape: Circle) -> float:
         return math.pi * shape.radius ** 2
     ```
   - This directly matches this repo's existing house style for
     type-based polymorphism (see python-higher-order-functools) — this
     skill applies that same tool specifically through the OCP lens.

3. **Add new variants by registering, never by modifying**
   - Adding support for a new shape means writing a new dataclass and a
     new `@area.register` implementation — no existing code changes:
     ```python
     @dataclass(frozen=True)
     class Triangle:
         base: float
         height: float

     @area.register
     def _(shape: Triangle) -> float:
         return 0.5 * shape.base * shape.height
     ```
   - This is the literal functional-lite satisfaction of "open for
     extension, closed for modification": the original `area` function
     definition, and every previously-registered implementation, remain
     byte-for-byte unchanged when `Triangle` is added.
   - Callers depend only on the `area(shape)` function signature, never on
     which concrete variant they're holding — the same decoupling the
     book's `AreaCalculator` achieves by depending on the abstract `Shape`
     type, achieved here by depending on a function name instead of a
     class.

4. **Know when NOT to use singledispatch — the expression-problem tradeoff**
   - Be explicit about the real tradeoff at play, often called the
     expression problem: subclass polymorphism (and `singledispatch`)
     makes it cheap to add new *types* without touching existing code, but
     expensive to add a new *operation* (you'd need to touch every type).
     Pattern matching over a closed set of dataclasses is the mirror
     image: cheap to add a new *operation* (write one new function with a
     `match`), expensive to add a new *type* (every existing `match`
     function needs a new `case`).
   - Recommend `singledispatch` (satisfying strict OCP) when the set of
     variants is genuinely expected to grow over the system's life, and
     new variants are more likely to appear than new operations on the
     existing variants.
   - Recommend structural pattern matching instead (see
     python-pattern-matching) when the set of variants is small and
     genuinely closed — e.g., a fixed protocol's message types, or a
     domain concept with an inherently bounded set of states — where
     readability of a single, co-located `match` block outweighs strict
     closed-for-modification adherence. In this case, adding a new
     variant *will* require editing every relevant `match` function, and
     that's an accepted, deliberate tradeoff, not an oversight.
   - Don't default to one or the other out of habit — name which property
     (extensibility to new types, or extensibility to new operations)
     actually matters more for the specific piece of logic before
     choosing.

---

## Decision points and guidance

- **Is the set of variants expected to grow?** If yes, use
  `singledispatch` — that's what actually satisfies OCP here. If the set
  is fixed and small, pattern matching is a reasonable, simpler choice.
- **Is a new subclass being proposed to add behavior for a new case?**
  Redirect to a new dataclass + a new `@operation.register` implementation
  instead.
- **Does adding this variant require touching existing code?** If using
  `singledispatch`, the answer should be no — if it's not, something's
  wrong with how the dispatch is structured (e.g., logic embedded in the
  default case that should be variant-specific).
- **Is an `isinstance()`/`if-elif` chain growing every time a new variant
  is added?** That's the exact code smell OCP addresses — replace it with
  `singledispatch` (or pattern matching, per the tradeoff above), not with
  another `elif` branch.

---

## Quality criteria

A strong OCP-satisfying design should ensure that:

- **variants are independent dataclasses**, never a subclass hierarchy,
- **the varying operation is a `singledispatch` function**, with per-type
  behavior registered rather than embedded in a growing conditional,
- **adding a new variant touches zero existing code** when
  `singledispatch` is the chosen tool,
- **the singledispatch-vs-pattern-matching choice is deliberate**, named
  explicitly in terms of which extensibility direction (new types vs. new
  operations) matters more for that specific piece of logic.

---

## Example prompts

- "This book example uses an abstract `Shape` class with subclasses —
  reformulate it using dataclasses and `singledispatch`."
- "We keep adding `elif isinstance(...)` branches every time a new
  variant shows up — help me refactor this to be genuinely open for
  extension."
- "Should this use `singledispatch` or `match`/`case` — which one
  actually fits how this data is going to evolve?"

---

## Related guidance

Use this tool alongside:

- python-higher-order-functools
- python-pattern-matching
- python-clean-architecture-single-responsibility
- python-clean-architecture-interface-segregation
