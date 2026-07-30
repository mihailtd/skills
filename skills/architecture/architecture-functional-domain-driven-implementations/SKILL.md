---
name: architecture-functional-domain-driven-implementations
description: Guides the implementation of domain models in a functional style, mapping domain storytelling work objects to types and activities to functions for safer state transitions.
---

# Functional Domain-Driven Implementations

This skill helps architects and developers implement domain models in a functional
style. It connects Domain Storytelling outcomes to functional programming
concepts such as typed work objects, algebraic data types, and state-transition
functions.

---

## When to use this skill

Use this skill when you need to:

- Turn domain stories into a functional domain model.
- Map business work objects to type-safe domain types.
- Implement activities as functions that transform state.
- Use the type system to prevent invalid domain states.
- Model entity lifecycle transitions explicitly in code.

---

## Outcome

Produce a functional domain model that:

- represents work objects as distinct types,
- implements domain activities as pure functions,
- encodes entity state with specific state types,
- uses algebraic data types to make illegal states unrepresentable,
- enforces business rules at compile time through type-driven transitions.

---

## Step-by-step workflow

1. **Extract work objects from the domain story**
   - Identify each domain object or value object from the story.
   - Model those objects as types in the functional language.
   - Prefer small, precise types for currency, term, amount, interest, and similar
     business concepts.

2. **Model activities as functions**
   - Translate each domain activity into a function signature.
   - Functions should accept typed inputs and return typed outputs.
   - Keep functions focused on a single domain behaviour.

3. **Define state-specific domain types**
   - Identify each valid state an entity can inhabit.
   - Create distinct types for each state, such as `FilledOutContract`,
     `CalculatedContract`, `OfferedContract`, and `SignedContract`.
   - Avoid a single mutable object that can hold invalid combinations of data.

4. **Use algebraic data types for entities**
   - Combine state-specific types into a single sum type that represents the
     entity as exactly one valid state.
   - This makes the domain model explicit and exhaustively typed.

5. **Implement transitions as functions**
   - Define functions that take a specific state type as input and produce the
     next valid state type.
   - For example, a `calculateInstallment` function takes a `FilledOutContract`
     and returns a `CalculatedContract`.
   - The type system prevents invalid transitions, such as calculating an already
     signed contract.

6. **Leverage the type system for safety**
   - Use the compiler to catch invalid state combinations and illegal transitions.
   - Keep domain rules encoded in function signatures and state types.
   - This reduces runtime errors and clarifies business intent.

7. **Iterate with domain experts**
   - Review the typed model with domain experts to ensure the states and
     transitions mirror the real business processes.
   - Refine types and functions as new story details emerge.

---

## Decision points and guidance

- **Use types for work objects:** Model business nouns as domain-specific types,
  not generic primitives.
- **Keep activities pure:** Domain functions should transform inputs into outputs
  without hidden side effects when possible.
- **State-driven entities:** Represent each entity state separately to make
  illegal states unrepresentable.
- **Favor algebraic data types:** Use sum types to model an entity as one of many
  valid states.
- **Enforce transitions:** Only allow state transitions that match the domain
  story behaviour.

---

## Quality criteria

A strong functional domain implementation should be:

- **Type-safe:** invalid states and transitions are blocked by the type system.
- **Expressive:** domain concepts are clearly represented by types and functions.
- **Predictable:** each activity maps to a well-defined function.
- **Aligned:** the implementation follows the domain story structure.
- **Reviewable:** domain experts can read the model and verify the states and
  transitions.

---

## Example prompt

- "Show me how to model a leasing contract domain using state-specific types
  and functions in a functional programming language."
- "Explain how to use algebraic data types to make illegal contract states
  unrepresentable."
- "Describe how to map domain storytelling actors, activities, and work objects to
  functional types and transition functions."

---

## Related guidance

This skill complements:

- Architecture Domain Storytelling
- Architecture Scenario-Based Modelling
- Business Analysis Scope
- Business Analysis Domain Storytelling + User Story Mapping