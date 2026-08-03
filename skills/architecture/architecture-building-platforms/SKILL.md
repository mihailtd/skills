---
name: architecture-building-platforms
description: Instructs the AI assistant to recognize when a product is (or is becoming) a platform — one that must appeal to two or more discrete customer sets, including developers who build on it directly — and to guide architecture decisions toward extensible, recomposable building blocks instead of a single-purpose, tightly coupled design.
---

# Building Platforms Instructions

When supporting architecture design, use this tool to help teams recognize
platform dynamics early, understand why architecture has an outsized impact
on platforms specifically, and design building blocks that stay easy to
recombine as the product's audience and use cases grow.

---

## Purpose

This tool helps the AI assistant by:

- identifying when a product is, or is at risk of becoming, a platform —
  appealing to two or more discrete customer sets rather than one,
- explaining why architecture is *the product* for a platform's developer
  audience, not an implementation detail one step removed from their
  experience,
- recommending a "build for recombination" design posture — components
  that depend minimally on their current relationships and connect easily
  into new ones,
- framing the ecosystem/ ecosystem feedback-loop rationale behind platform
  investment, so architectural effort here is understood as a business bet,
  not gold-plating,
- warning against treating platform-readiness as a later, bolt-on concern
  once a product's trajectory already points that way.

---

## Expected outcome

As the AI, your response should help teams adopt an architecture that:

- distinguishes platform products (multiple discrete customer sets, one of
  which builds directly on the architecture) from single-audience products,
- treats extensibility and composability as first-class architectural
  properties whenever platform dynamics are present or plausible,
- designs components as building blocks — loosely coupled, minimally
  dependent on their current wiring, easy to recombine into new
  configurations,
- anticipates the natural evolution path from single-purpose product to
  platform (e.g., an application gaining macros, then plugins) instead of
  being surprised by it,
- avoids over-engineering for platform flexibility when there's no credible
  developer audience or extension use case on the horizon.

---

## Instructions for the AI

1. **Identify whether the product is (or is becoming) a platform**
   - Ask whether the product appeals to two or more *discrete* customer
     sets — not just different user roles, but genuinely different kinds of
     stakeholders with different needs from the system (e.g., end users who
     consume functionality versus developers who extend it).
   - Treat "developers who build on this directly" as the defining signal of
     a platform, distinct from ordinary end-user role differentiation (like
     regular users versus administrators, who are still both "customers" of
     the same functional product).
   - Watch for the natural evolution path: a single-purpose product (e.g.,
     a word processor) often grows macro support, then plugin support, then
     a genuine developer ecosystem. Flag this trajectory early rather than
     treating platform needs as a sudden, unplanned pivot.

2. **Explain why architecture matters more for platforms**
   - Make clear that for a platform's developer audience, the architecture
     *is* the product — developers interact directly with the building
     blocks, not just the functional outcome those blocks produce.
   - Contrast this with a pure end-user product, where a good architecture
     helps the product perform well, but users experience the *function*,
     not the architecture itself.
   - Frame platform investment as a business bet on an ecosystem feedback
     loop between users and developers — explain that this is why platform
     owners sometimes actively invest in (or even pay) developers to target
     the platform, since developers are what kick-starts that loop.

3. **Design for recombination, not just for the current use case**
   - Recommend building components that depend as little as possible on
     their *current* relationships to other components.
   - Recommend making it cheap and low-risk to connect a component to *new*
     relationships later — this is the concrete design property that makes
     a component a genuine "building block" rather than a fixed part of one
     assembly.
   - Point out that this same property pays off even if the product never
     becomes a full plugin platform — it's what makes ordinary product
     evolution and feature recombination cheaper too.

4. **Calibrate the investment to actual platform likelihood**
   - Advise against speculative extensibility for products with no
     plausible second customer set or developer audience — recombinable
     building blocks are a deliberate design investment, not a default to
     apply everywhere regardless of need.
   - When platform potential is real but not yet proven, recommend the
     "every product is a platform, or will be eventually" planning stance:
     treat the building-block property as a feature worth investing in
     incrementally, not as something to defer entirely until plugins are
     explicitly requested.

---

## AI decision guidance

When generating platform-architecture guidance, keep these principles in mind:

- **Two discrete customer sets is the test, not just multiple user roles:**
  don't call something a platform just because it has both regular users
  and admins — look for a genuinely separate audience (developers) with a
  different relationship to the system.
- **Architecture is the product for developers:** treat API design, component
  boundaries, and extension points with the same rigor a UX team would apply
  to the end-user-facing product.
- **Optimize for recombination, not just correctness:** a well-functioning
  component that's hard to reconnect into a new configuration is not yet a
  good building block.
- **Plan for evolution, don't assume it:** the "eventually a platform"
  mindset is about resilience to a plausible future, not a mandate to build
  a plugin architecture for every product on day one.
- **Weigh the ecosystem investment honestly:** platform strategy is a bet on
  a feedback loop that may or may not materialize — architectural
  flexibility has a cost, and that cost should be justified by real signal,
  not applied reflexively.

---

## Success criteria

A strong platform-architecture response should help teams achieve:

- **clear platform recognition:** an explicit answer to "is this a platform,
  or heading toward one" grounded in the two-discrete-customer-sets test,
- **developer-grade architecture:** component boundaries and extension
  points designed with the same care as end-user-facing functionality,
- **recomposable building blocks:** components that are cheap to rewire into
  new relationships, not just correct in their current one,
- **evolution readiness:** a design that accommodates the common
  single-purpose-product-to-platform trajectory without a disruptive
  rewrite,
- **calibrated investment:** platform-readiness effort that's proportional
  to actual evidence of a second customer set, not applied by default.

---

## Example prompts for the AI

- "Help me figure out whether the internal tool we're building is turning
  into a platform, and what that means for how we should architect it."
- "We're adding a plugin system to our application — how should we design
  the core components so they're easy for third-party developers to extend?"
- "Explain why we should treat our API design with the same rigor as our
  end-user UI."

---

## Related guidance

Use this tool alongside:

- architecture-standards-adoption
- architecture-reduce-risk
- architecture-domain-storytelling
- architecture-composition
- architecture-decomposition
