---
name: architecture-building-with-failure-in-mind
description: Guides the design of systems that handle dependency failures through graceful degradation, graceful backoff, and failing early.
---

# Building with Failure in Mind

This skill helps architects design systems that anticipate failures and respond
with predictable, understandable, and reasonable behavior. It emphasizes graceful
degradation, graceful backoff, and failing as early as possible.

---

## When to use this skill

Use this skill when you need to:

- design services that remain useful even when dependencies fail,
- avoid cascading service failures across distributed systems,
- preserve API contracts during degraded operation,
- provide alternative user experiences when critical dependencies fail,
- fail fast and with clear error semantics when a request cannot be fulfilled.

---

## Outcome

Produce a design approach that:

- treats dependency failures as expected events,
- defines planned responses for failure scenarios,
- keeps service behavior predictable and contract-safe,
- provides reasonable fallback experiences when possible,
- fails early and clearly when no acceptable alternative exists.

---

## Instructions for the AI

1. **Evaluate dependencies**
   - For every component or external service, ask: what happens if it fails?
   - Determine whether the system can tolerate the failure with partial
     functionality, needs an alternative path, or must fail fast.

2. **Design graceful degradation**
   - Recommend ways for the system to continue performing core work without the
     failing component.
   - Preserve value for the consumer by showing partial results, cached data, or
     simplified functionality rather than failing the entire request.
   - Use examples such as displaying product details without images or cached
     prices when a pricing service is unavailable.

3. **Design graceful backoff**
   - Identify cases where the request cannot fully succeed but the experience can
     still redirect to a useful alternative.
   - Recommend alternative workflows or fallback content, such as showing popular
     products instead of an empty error page.
   - Ensure the alternative behavior remains reasonable for the situation.

4. **Enforce failing early**
   - Recommend strict validation and boundary checks at the start of workflows.
   - If a request is invalid or lacks critical resources, fail immediately with a
     clear error instead of continuing work.
   - Emphasize resource conservation, faster response, and simpler diagnostics.

5. **Keep responses predictable and contract-safe**
   - Ensure any failure response conforms to the agreed API format.
   - Avoid returning malformed or unexpected data because an upstream dependency
     broke.
   - Recommend clear error codes and messages that reflect the real failure.

6. **Use “appropriate action” as risk mitigation**
   - For each failure mode, suggest the most appropriate action: degrade,
     back off, or fail early.
   - Frame the choice based on predictability, understandability, and reasonableness.

---

## Decision points and guidance

- **Preserve API contracts:** never violate the response format or schema just
  because an upstream dependency failed.
- **Prefer graceful degradation:** if the system can still provide useful output,
  degrade functionality rather than fail completely.
- **Use graceful backoff when needed:** provide an alternate value or workflow
  when the primary flow is unavailable.
- **Fail early when necessary:** stop processing invalid or hopeless requests
  immediately.
- **Make failure behavior reasonable:** choose responses that make sense for the
  consumer and the business context.

---

## Quality criteria

A good failure-aware design should be:

- **Predictable:** responses are planned and consistent.
- **Understandable:** behavior remains within agreed contracts.
- **Reasonable:** fallback actions make sense for the failure mode.
- **Resilient:** the overall system avoids cascading failures.
- **Clear:** errors are surfaced early with straightforward diagnostics.

---

## Example prompt

- "Explain how to design an API so it can degrade gracefully when an external
  pricing service is unavailable."
- "Show me how to choose between graceful degradation, graceful backoff, and
  failing early for a particular dependency failure."
- "Describe how to keep a service predictable and contract-safe when a backend
  dependency breaks."

---

## Related guidance

This skill complements:

- architecture-domain-storytelling
- architecture-scenario-based-modelling
- business-analysis-scope
