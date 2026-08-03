---
name: architecture-flexibility-complexity-tradeoffs
description: >-
  Instructs the AI assistant to treat API flexibility and complexity as a single tradeoff axis rather than a free improvement — abstracting away a concrete dependency shifts complexity to the client (a hybrid "abstract, but ship a sensible default implementation" design is usually the sweet spot), while hooks and listener extensibility patterns buy the most flexibility at the steepest cost: hooks require treating caller-supplied code as untrusted plus a deliberate sync-vs-async execution-model decision, and listeners are only safe when the state handed to them is immutable.
---

# Flexibility vs. Complexity Tradeoffs Instructions

When supporting API or component design, use this tool to evaluate a request
for more flexibility or extensibility as a real cost that gets relocated, not
removed — and to apply the right mitigation for whichever point on the
flexibility axis a design lands on: dependency abstraction, a hooks API, or a
listener API.

---

## Purpose

This tool helps the AI assistant by:

- treating "make this more flexible/extensible" as a design decision with a
  real, often underestimated cost, not a free improvement — every added
  degree of flexibility moves complexity somewhere (the client, the
  execution model, ongoing operations) rather than eliminating it,
- distinguishing three points on a single flexibility axis, each with a
  different cost shape: abstracting away a concrete dependency (lowest cost,
  least flexible), a hooks API (synchronous extension points), and a
  listener API (asynchronous extension points),
- recommending the specific mitigation each point needs: a hybrid
  abstract-with-a-default-implementation design for dependency abstraction;
  defensive failure-handling plus a deliberate sync-vs-async execution-model
  decision for hooks; immutable state propagation for listeners,
- pushing back on defaulting to the most generic, most flexible extension
  mechanism when a narrower design would satisfy the actual, currently known
  use cases.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- identifies where on the flexibility-complexity axis a proposed design
  sits, and names the specific cost that flexibility buys (client-side
  burden, defensive-programming burden, execution-model complexity,
  operational overhead) rather than treating "more flexible" as an
  unqualified good,
- defaults to a hybrid design when abstracting a concrete dependency —
  expose the abstraction, but ship a default implementation for the common
  case — rather than forcing every consumer to supply their own,
- when a hooks API is genuinely warranted, addresses both of its costs
  together: hook code is treated as untrusted (guarded against arbitrary
  exceptions), and the sync-vs-async execution model is a deliberate,
  documented choice rather than an accident,
- when a listener API is genuinely warranted, insists on immutable state
  being what's propagated, since the host component can't reason about or
  control what listener code does with a mutable reference,
- doesn't recommend the most flexible/generic pattern by default — matches
  the extensibility mechanism to actually known use cases, and revisits only
  when a real new requirement demands more.

---

## Instructions for the AI

1. **Recognize that flexibility doesn't remove complexity, it relocates it**
   - When a component becomes more abstract, pluggable, or extensible, the
     complexity that used to live inside it doesn't disappear — it moves to
     whoever configures or extends it: the client providing an
     implementation, the caller writing hook/listener code, the team
     operating a new execution model.
   - Before recommending a more flexible design, name explicitly where the
     complexity is moving to, and evaluate whether that's actually a good
     trade for the people who will bear it — not just whether it makes the
     component itself look cleaner.

2. **Prefer the hybrid pattern when abstracting away a concrete dependency**
   - Abstracting a concrete third-party dependency behind an interface is the
     lowest-cost point on the flexibility axis: it decouples the component
     from one specific implementation, but every client now needs to supply
     (or be given) a concrete implementation to actually use the component.
   - Default to a hybrid design: expose the abstraction for consumers who
     genuinely need a different implementation, but ship a default,
     ready-to-use implementation for the common case, so most callers never
     have to think about the abstraction at all. Reserve "abstraction with no
     default" for cases where there genuinely is no sensible default (e.g.
     credentials or environment-specific configuration that can't be
     guessed).

3. **If a hooks API is warranted, budget for both of its costs together**
   - **Untrusted caller code:** a hooks API hands control to caller-supplied
     code running inside your component's own execution path. Treat that
     code as untrusted — guard against it raising any exception, and don't
     let a failure in a hook take down or corrupt the host component's own
     processing.
   - **The execution-model decision is not free either way:** if hooks run
     synchronously, a slow or blocking hook directly extends your
     component's own latency and can violate its SLA — you're exposed to
     work you don't control. If you make hook execution asynchronous to
     avoid that, you now own a new piece of infrastructure: a dedicated
     thread/task pool to size and monitor, and a happens-before relationship
     between processing phases that limits how much that parallelism can
     actually hide the added latency — asynchronous hook execution reduces
     the *risk* to your own SLA, it does not reduce the *total* latency cost
     to zero.
   - Make the sync-vs-async choice for hook execution explicit and
     documented — don't let it default to whichever was easiest to implement
     first.

4. **If a listener API is warranted, propagate only immutable state**
   - A listener API avoids the synchronous-blocking cost of hooks by
     emitting signals asynchronously instead of pausing your own processing
     for the listener to run — but this doesn't remove the
     untrusted-caller-code concern, it changes its shape.
   - Once state is handed to a listener, there's no visibility into or
     control over what happens to it — if that state is mutable, a listener
     can modify it in ways that corrupt the host component's subsequent
     processing, or that surprise other listeners observing the same event.
   - Always pass immutable state to listeners. This is not a stylistic
     preference — it's the specific property that makes a listener API safe
     to expose at all, given listener code's behavior can't be reasoned
     about or constrained.

5. **Match the extensibility mechanism to actual known use cases, not maximum generality**
   - The temptation, when a use case isn't fully known yet, is to reach for
     the most generic mechanism (a hooks or listener API) so "anything is
     possible later." Push back on this — maximum generality also means
     maximum cost (points 3 and 4), paid immediately, for flexibility that
     may never actually be used.
   - Recommend starting at the least flexible point on the axis that
     satisfies the currently known use cases (a concrete implementation, or a
     narrow abstraction with a default), and moving toward hooks/listeners
     only when a real, specific requirement demands that level of
     extensibility — not preemptively.

---

## AI decision guidance

When generating flexibility/extensibility guidance, keep these principles in
mind:

- **Every point on the flexibility axis has a cost, and that cost doesn't
  disappear when you add flexibility** — it relocates to the client, to
  defensive-programming burden, or to a new execution-model/operational
  responsibility. Name where it lands before recommending a design.
- **Default to a hybrid abstraction-with-a-default-implementation** for
  dependency abstraction — pure abstraction with no default pushes
  unnecessary burden onto every caller.
- **A hooks API's two costs — untrusted caller code and the sync-vs-async
  execution model — must be addressed together**, not just one of them;
  handling exceptions without deciding the execution model (or vice versa)
  is an incomplete design.
- **A listener API is only safe with immutable state** — this isn't optional
  hardening, it's the property that makes the pattern viable at all.
- **Don't recommend the most generic/flexible extensibility mechanism
  preemptively** — match it to known use cases and let genuinely new
  requirements justify moving further along the axis.

---

## Success criteria

A strong response should ensure that it:

- **identifies which point on the flexibility-complexity axis** a design
  sits at, and names where the corresponding complexity actually lands,
- **defaults dependency-abstraction recommendations to the hybrid pattern**
  (abstraction plus default implementation) unless no sensible default
  exists,
- **addresses both hooks-API costs together** — untrusted-code handling and
  an explicit sync-vs-async execution-model decision,
- **requires immutable state propagation** for any listener API
  recommendation,
- **doesn't recommend maximum flexibility/genericity as a default** when the
  actual known use cases are narrower.

---

## Example prompts for the AI

- "Should we make this component's storage backend pluggable, or just
  support Postgres directly?"
- "We want to let other teams hook into our processing pipeline — what do we
  need to think about?"
- "Is it safer to expose a hooks API or a listener API for this extension
  point?"
- "Someone wants to add a generic plugin system 'in case we need it later' —
  is that a good idea?"

---

## Related guidance

Use this tool alongside:

- architecture-simplicity — the general principle (few, powerful
  abstractions over speculative future-proofing) this skill applies
  specifically to API extensibility mechanisms.
- architecture-code-duplication-tradeoffs
- architecture-inheritance-coupling-tradeoffs
- architecture-exception-design-and-anti-patterns — the same
  untrusted-caller-code discipline behind this skill's hooks-API guidance
  (point 3), applied generally to exception design.
- architecture-decomposition
- architecture-configuration-surface-tradeoffs — the same source book's
  flexibility-vs-maintenance axis, applied specifically to a dependency's
  configuration surface (passthrough vs. ownership) rather than API
  extensibility mechanisms.
- architecture-third-party-defaults-and-concurrency — the same "cost
  relocates, it doesn't disappear" principle applied to wrapping a blocking
  third-party library in an async facade.
