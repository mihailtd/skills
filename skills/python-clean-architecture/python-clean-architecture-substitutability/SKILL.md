---
name: python-clean-architecture-substitutability
description: Instructs the agent to apply the Liskov Substitution Principle as substitutability of functions sharing a Callable type, not substitutability of subclasses — any function matching a given Callable signature must honor the same behavioral contract (same meaning of inputs/outputs, same error semantics) as every other function matching that signature. Reformulates the classic Vehicle/inheritance violation as composition via a passed-in "operation" function, not a base-class contract.
---

# Python Clean Architecture: Substitutability, Functional-Lite

The Liskov Substitution Principle (LSP) says any subclass should be usable
anywhere its base class is expected, without breaking the caller's
assumptions. Functional-lite has no subclass hierarchies for business
logic, so LSP's mechanism doesn't directly apply — but its underlying
discipline does, at a different unit: **any function matching a shared
`Callable` type must honor the same behavioral contract as every other
function matching that type.** This skill covers how to spot and fix
violations of that discipline, using the classic
fuel-vehicle/electric-vehicle example reformulated as composition over a
passed-in function instead of inheritance.

---

## When to use this skill

Use this skill when you need to:

- design two or more functions meant to be interchangeable behind the same
  `Callable` type,
- translate an LSP violation example (a subclass that quietly changes the
  meaning of an inherited method) into this repo's style,
- review code where swapping one passed-in function for another
  (matching the same type signature) produces subtly wrong results,
- decide whether two operations are similar enough to share a `Callable`
  type, or different enough that forcing a shared signature is the actual
  bug.

---

## Outcome

Produce substitutable, functional-lite code that:

- expresses a swappable behavior as a `Callable` type (see
  python-clean-architecture-dependency-inversion), with every function
  matching that type honoring the same meaning for its inputs, outputs,
  and error conditions,
- uses composition — a dataclass holding a reference to which function
  implements its variable behavior — instead of inheritance to model
  "this kind of thing behaves differently depending on which variant it
  is,"
- catches the case where two operations only *look* similar enough to
  share a signature, but actually differ in a way that breaks callers when
  swapped,
- lets calling code work correctly with any function matching the shared
  type, without knowing or caring which specific one it received.

---

## Instructions for the AI

1. **Recognize an LSP violation as a broken behavioral contract, not a broken class hierarchy**
   - Translate the book's `Vehicle`/`ElectricCar` example: an
     `ElectricCar(Vehicle)` subclass that overrides `consume_fuel` to mean
     something subtly different (energy consumption at a different rate
     and unit) while keeping the same method name and signature — callers
     written against `Vehicle` silently get wrong results when handed an
     `ElectricCar`.
   - In functional-lite terms, the equivalent mistake is: two functions
     sharing a `Callable[[float], float]` type where one means "liters of
     fuel consumed" and the other means "kWh of energy consumed" — the
     *type* matches, but the *meaning* doesn't, so code that's generic
     over the shared type produces misleading results when the
     implementation is swapped.

2. **Fix it with composition over a passed-in function, not inheritance**
   - Reformulate the book's `PowerSource` abstraction — which still uses
     an ABC and subclassing — as a `Callable` type and independent plain
     functions instead:
     ```python
     from typing import Callable
     from dataclasses import dataclass

     # The "abstraction" — a function type, not a base class
     ConsumeFn = Callable[[float, float], float]  # (level, distance) -> amount_consumed

     def consume_fuel(level: float, distance: float) -> float:
         consumed = distance / 10  # 10 km per liter
         if level - consumed < 0:
             raise ValueError("Not enough fuel to cover the distance")
         return consumed

     def consume_charge(level: float, distance: float) -> float:
         consumed = distance / 5  # 5 km per kWh
         if level - consumed < 0:
             raise ValueError("Not enough charge to cover the distance")
         return consumed
     ```
   - Model the "vehicle" as a dataclass holding its current level and a
     reference to which consume function applies — composition, not
     inheritance:
     ```python
     @dataclass(frozen=True)
     class Vehicle:
         level: float
         consume: ConsumeFn

     def drive(vehicle: Vehicle, distance: float) -> tuple[Vehicle, float]:
         consumed = vehicle.consume(vehicle.level, distance)
         return replace(vehicle, level=vehicle.level - consumed), consumed
     ```
   - Both `consume_fuel` and `consume_charge` now genuinely honor the same
     contract: given a level and a distance, return the amount consumed
     (in whatever unit that vehicle's level is denominated in), or raise
     `ValueError` if there isn't enough — `drive` works correctly with
     either, because both implementations mean the same thing by "consume,"
     not just because they share a type signature.

3. **Verify substitutability by checking meaning, not just type signature**
   - When two functions share a `Callable` type, explicitly check: do they
     mean the same thing by their inputs and outputs? Do they raise the
     same kind of error under the same conditions? Do they have the same
     side-effect profile (both pure, or both impure in compatible ways)?
   - Matching type signatures is necessary but not sufficient for
     substitutability — the book's `ElectricCar.consume_fuel` had the
     exact right signature and still violated the caller's assumptions,
     because the *meaning* of "fuel consumed" changed silently.
   - Treat a passed-in function that requires the caller to know which
     specific implementation it received (to interpret the result
     correctly) as a substitutability violation, exactly as much as a
     subclass that requires callers to know which concrete type they're
     holding.

4. **Recognize when two operations shouldn't share a type at all**
   - If forcing two operations into the same `Callable` signature requires
     awkward unit conversions, inconsistent error handling, or a comment
     explaining "this one behaves a bit differently," that's a signal the
     operations don't actually belong under one shared abstraction — give
     them different, more honestly-named types instead of forcing a false
     substitutability.
   - This mirrors the book's own diagnosis: the root problem wasn't
     `ElectricCar`'s implementation — it was forcing an electric vehicle's
     genuinely different consumption model into the same interface as a
     fuel vehicle's. The fix isn't a cleverer shared interface; it's
     recognizing where the shared abstraction should honestly end.

---

## Decision points and guidance

- **Do two functions sharing a `Callable` type mean the same thing by their
  inputs, outputs, and errors?** If not, that's a substitutability
  violation, regardless of whether the type signatures match exactly.
- **Does calling code need to know which specific function it received to
  interpret the result correctly?** If so, the shared type is providing
  false confidence — fix the contract or split the type.
- **Is forcing two operations into one `Callable` type requiring
  workaround logic (unit conversions, special-casing) at call sites?**
  That's a sign the operations shouldn't share a type at all.
- **Is composition (a function reference on a dataclass) being modeled
  instead of inheritance?** Confirm the "variant" behavior is passed in as
  a function, not encoded as a subclass overriding a method.

---

## Quality criteria

A strong substitutability implementation should ensure that:

- **every function sharing a `Callable` type honors the same behavioral
  contract**, not just the same type signature,
- **variant behavior is modeled via composition** (a function passed into
  a dataclass or another function), never via inheritance overriding a
  method,
- **calling code works correctly with any conforming function**, without
  needing to know which specific implementation it received,
- **operations that don't genuinely share a contract are given distinct
  types**, rather than being forced together under one signature.

---

## Example prompts

- "This book example has an `ElectricCar` subclass that breaks
  `Vehicle`'s contract — reformulate it using composition and a passed-in
  function instead of inheritance."
- "These two functions have the same type signature but behave
  differently in a way that breaks callers when swapped — is that an LSP-
  style violation?"
- "Should these two operations share a `Callable` type, or are they
  different enough that forcing a shared signature is the real bug?"

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-dependency-inversion
- python-clean-architecture-open-closed
- python-clean-architecture-interface-segregation
