---
name: architecture-composition
description: Instructs the AI assistant to design decomposed elements with recomposition in mind from the start — keeping the relationships between pieces simple and efficient (matching interaction granularity to interface cost, e.g. batches over record-by-record across a network boundary), standardizing wiring mechanisms, and, for platform designs, balancing standards-as-constraint against the flexibility needed for unanticipated combinations.
---

# Composition Instructions

When supporting architecture design, use this tool to help teams treat
composition as inseparable from decomposition — anticipating how the pieces
will recombine while still deciding how to divide them, keeping the
relationships between elements simple and efficient, standardizing the
mechanisms used to wire them together, and, for platform-style designs,
deliberately balancing the constraint standards provide against the
flexibility needed for combinations the designer didn't anticipate.

---

## Purpose

This tool helps the AI assistant by:

- treating decomposition and composition as two sides of the same
  activity — a breakdown that can't be reassembled into a working whole
  isn't a usable decomposition, so recomposition should be anticipated
  during decomposition, not solved afterward,
- pushing for simple, efficient relationships between decomposed elements,
  since a convoluted set of relationships produces a complex solution
  regardless of how clean the individual pieces are,
- matching interaction granularity to the real cost of the interface
  between elements — e.g., recognizing when a record-by-record interaction
  pattern won't hold up across a boundary meant to operate at scale, and
  batch/stream-based interaction is needed instead,
- weighing interface placement against real latency costs — a cross
  process or cross-machine boundary is a materially different composition
  decision than a boundary within the same process,
  and recommending standardization of the wiring mechanisms used across a
  system to avoid paying repeated translation costs between heterogeneous
  mechanisms,
- for platform designs specifically, framing the central design tension as
  standards-as-constraint (so building blocks reliably fit together) versus
  flexibility (so genuinely new, unanticipated combinations remain
  possible) — and treating "anticipating the unanticipated" as the defining
  challenge of platform composition.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- designs decomposition and composition together — asking "how will these
  pieces recombine" at the same time as "how should we divide this
  problem," not as a separate, later step,
- keeps the relationships between decomposed elements as simple as
  possible, treating a convoluted set of interfaces as a real design defect
  even when the individual elements are each well-designed,
- checks whether the granularity of interaction between elements matches
  the actual performance requirement — flagging record-by-record designs
  that won't scale, and recommending batch- or stream-based interaction
  where warranted,
- treats interface boundaries as real cost decisions — weighing whether a
  cross-process or cross-machine boundary's latency is acceptable, or
  whether a tighter-grained decomposition (e.g., libraries or classes
  within one service) is needed instead,
- standardizes the mechanisms used to wire elements together across a
  system, reducing the translation overhead that a heterogeneous mix of
  wiring approaches creates,
- for systems that are, or may become, platforms (see
  architecture-building-platforms), explicitly reasons about which
  interfaces should be constrained by standards to guarantee composability,
  and which aspects should stay flexible to allow combinations the designer
  didn't originally plan for.

---

## Instructions for the AI

1. **Design composition and decomposition together**
   - Make explicit that a decomposition which can't be recomposed into a
     working solution isn't functional — anticipate how the resulting
     pieces will be stitched back together while deciding how to divide
     the problem in the first place, not as an afterthought once the
     pieces already exist.

2. **Keep inter-element relationships simple**
   - Treat overly convoluted relationships between decomposed elements as a
     real design cost — even a clean decomposition of individual elements
     produces a complex, hard-to-maintain solution if putting them back
     together is difficult to do, hard to get right, and hard to keep
     working.
   - When reviewing a proposed decomposition, explicitly assess the
     relationships it implies, not just the elements themselves.

3. **Match interaction granularity to real interface cost and scale**
   - When elements interact (e.g., logic components operating on data
     components), check whether the interaction pattern matches the actual
     performance requirement. A record-by-record pattern keeps the simple
     case simple, but often breaks down when logic needs to operate at
     scale across many records — recommend batch- or stream-based
     interaction in that case instead of, or in addition to, record-by-
     record.
   - Weigh interface placement against latency cost directly: a
     decomposition that places elements in separate services introduces a
     cross-process (and possibly cross-machine) boundary with real latency
     implications. If that latency is unacceptable, recommend a
     finer-grained decomposition (libraries or classes within a single
     service) that can reduce interaction cost by orders of magnitude.

4. **Standardize the mechanisms used to compose elements**
   - Recommend that a system standardize on a minimally sufficient set of
     wiring mechanisms (function calls, network requests, message-passing)
     rather than letting many different mechanisms coexist.
   - Point out that a heterogeneous mix of composition mechanisms requires
     ongoing translation effort between them, and that constraining each
     component's design to a shared set of mechanisms removes that
     recurring cost.

5. **For platform-style designs, balance standards-as-constraint against flexibility**
   - When the system being designed is (or may become) a platform — one
     whose assembly is left to the user or developer rather than fully
     precomposed by the designer (see architecture-building-platforms) —
     recognize that its composition challenge is fundamentally different:
     it must support combinations the designer never explicitly planned
     for, not just the ones it does.
   - Recommend leaning more heavily on standards specifically as
     constraints in this setting — they're what makes it possible for
     independently built pieces to reliably fit together at all — while
     still preserving enough flexibility in the interfaces for genuinely
     new, useful combinations to emerge.
   - Frame "anticipating the unanticipated" as the core, unavoidable
     tension of platform composition design: too much constraint forecloses
     interesting combinations; too little constraint means the building
     blocks don't reliably fit together. Most of the platform design effort
     goes into finding the right balance for the specific system, not into
     eliminating the tension.

6. **Look for reuse opportunities the decomposition creates**
   - While decomposing a problem, look for elements whose interface could
     be modestly broadened or generalized to serve other parts of the
     system as well — this is the same underlying intuition behind shared
     libraries and other reusable components.
   - Watch for recurring subproblems across different parts of the system
     (e.g., multiple places needing text processing) — when a genuine
     pattern exists, even with imperfect overlap, treat it as an
     opportunity to create a single shared element rather than solving the
     same problem independently in each location. Be careful not to force
     this prematurely for superficially similar but actually distinct
     needs (see architecture-simplicity's caution on premature
     generalization).

---

## AI decision guidance

When generating composition guidance, keep these principles in mind:

- **Decomposition without a composition plan is incomplete:** always ask how
  the proposed pieces recombine, at the same time the breakdown itself is
  being evaluated.
- **Relationship complexity counts as design cost:** judge a decomposition
  by its composition difficulty, not just by how clean each individual
  piece looks in isolation.
- **Granularity should follow real interaction needs:** don't accept a
  record-by-record or a cross-process interface by default — check it
  against the actual scale and latency requirements.
- **Fewer wiring mechanisms, less translation cost:** push for
  standardized composition mechanisms across a system rather than tolerating
  a heterogeneous mix.
- **Platform composition is a deliberate constraint/flexibility balance:**
  don't treat "add more standards" or "add more flexibility" as
  unconditionally correct — reason about the specific tradeoff for the
  system at hand.

---

## Success criteria

A strong composition response should help teams achieve:

- **co-designed decomposition and composition:** a breakdown that was
  evaluated for how well it recombines, not just for how it divides,
- **simple inter-element relationships:** interfaces between elements that
  are straightforward to wire together and maintain,
- **scale-appropriate interaction granularity:** interaction patterns
  (record-by-record, batch, stream) chosen to match real performance needs,
- **cost-aware boundary placement:** interface boundaries placed with
  explicit awareness of the latency they introduce,
- **standardized wiring:** a minimal, consistent set of composition
  mechanisms used across the system,
- **balanced platform composability:** for platform-style systems, an
  explicit, reasoned tradeoff between standards-as-constraint and
  preserved flexibility for unanticipated combinations.

---

## Example prompts for the AI

- "We've decomposed this system into services, but putting them back
  together is getting complicated — help me evaluate whether the
  decomposition itself is the problem."
- "Our logic component talks to our data component one record at a time,
  and it's not keeping up at scale — how should we rethink that interface?"
- "We're building a platform and trying to decide how much to standardize
  versus how much flexibility to leave for third-party developers — help me
  reason through that tradeoff."

---

## Related guidance

Use this tool alongside:

- architecture-decomposition
- architecture-building-platforms
- architecture-standards-adoption
- architecture-simplicity
