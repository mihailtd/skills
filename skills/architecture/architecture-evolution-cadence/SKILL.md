---
name: architecture-evolution-cadence
description: Instructs the AI assistant to treat architectural-vision change as a rare, deliberate event rather than a frequent one — testing technology-driven change for real market/customer impact, weighing switching costs honestly against sunk expertise, and using a regular, calendar-aligned review cycle as both a decision point and a relief valve for ad hoc pressure to change.
---

# Evolution Cadence Instructions

When supporting architecture design, use this tool to help teams pace how
often the architectural vision itself changes — keeping it rare and
deliberate, testing technology-driven pressure to change against real
market impact, weighing switching costs honestly, and using a regular
review cycle to make evolution a considered process instead of a series of
ad hoc reactions.

---

## Purpose

This tool helps the AI assistant by:

- distinguishing evolving the *architectural vision* (rare, significant)
  from ordinary incremental delivery toward an existing vision (frequent,
  see architecture-incremental-delivery),
- warning that a frequently-moving target produces a chaotic, disorganized
  system no matter how well the day-to-day process is run — vision
  stability is a prerequisite for coherent incremental progress,
  not an obstacle to it,
- applying a market/customer-impact test to technology-driven change
  proposals, so novelty (microservices, a new database paradigm, etc.)
  isn't mistaken for justification,
- making switching costs concrete and honest — including the sunk,
  hard-to-see cost of a team's accumulated operational expertise in its
  current technology — before recommending a technology change,
  particularly the tricky "relevant but late" case,
- recommending a regular, calendar-aligned architectural review cycle that
  serves both as a deliberate checkpoint for evolution and as a relief
  valve that defers ad hoc "should we change this" pressure to a scheduled
  forum.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- treats a change to the architectural vision itself as a significant,
  infrequent event, and is suspicious of a vision that has shifted
  multiple times in a short period,
- tests any technology-driven change proposal against a concrete question:
  does this translate into real customer or market value, not just
  technical novelty,
- when a technology is relevant but arrived after the system already
  standardized on an alternative, weighs the switch honestly — quantifying
  what's already been invested in the current technology's operational
  maturity, not just comparing feature lists,
- recognizes that evolution is inevitable for any long-lived, successful
  system (every architecture eventually meets its limits) — the goal is
  pacing that evolution well, not preventing it,
- treats a well-organized, well-documented current architecture as the
  single most effective preparation for its own future evolution,
- recommends a regular review cadence, aligned with the organization's
  planning calendar, that both evaluates whether the vision still holds and
  absorbs ad hoc pressure to chase new technology outside that process.

---

## Instructions for the AI

1. **Keep the architectural vision itself rare to change**
   - Distinguish sharply between incremental work executed *toward* an
     existing vision (should happen constantly — see
     architecture-incremental-delivery) and a change *to the vision
     itself* (should happen rarely and deliberately).
   - If the vision appears to be shifting every few months, name this
     explicitly as a problem: a moving target turns even well-executed
     incremental steps into a random walk, producing a chaotic,
     disorganized system regardless of how disciplined day-to-day delivery
     is.

2. **Apply a market/customer-impact test to technology-driven change**
   - When a change is motivated by a new technology or platform rather than
     a product need, ask directly whether it translates into real customer
     or market value — not just whether it's newer, more elegant, or
     currently popular.
   - Use the contrast between changes with genuine market impact (e.g., a
     platform shift that creates real new user expectations and new
     product categories) and changes that are mostly technical novelty
     without a corresponding customer benefit (name the pattern —
     technology adopted primarily because it's new and exciting, without a
     clear customer-facing payoff) to calibrate how seriously to take the
     proposal.
   - Recommend against evolving the architecture to accommodate a
     technology shift that lacks this market/customer justification, even
     if the technology itself is legitimate and well-regarded.

3. **Weigh switching costs honestly, especially the "relevant but late" case**
   - When a new technology is a genuinely good fit but arrived after the
     system already committed to and matured on an alternative, treat this
     as the hard case it is rather than a simple feature comparison.
   - Make the sunk investment concrete: the current technology likely
     represents a large, mostly-invisible investment in tacit
     operational knowledge — how it behaves in production, its API and
     debugging quirks, how to deploy and keep it running — accumulated
     over real person-time, not just the visible code built on top of it.
   - Weigh this honestly against the new technology's promised benefit:
     switching means re-earning that same operational maturity from
     scratch on the new technology, with the added risk that the new
     technology may not pan out as expected. Set the bar for switching
     high enough to reflect both of these costs, not just the first.

4. **Treat evolution as inevitable but pace it deliberately**
   - Make clear that avoiding architectural evolution altogether isn't the
     goal — every architecture eventually meets limits its original design
     didn't anticipate, and hitting that limit is often a sign of success
     (the system outgrew its original remit), not failure.
   - Frame the actual goal as pacing evolution well: invest sufficiently
     when evolution genuinely happens, since a larger, more thorough
     investment in an upgrade should extend the time before the next one
     is needed — a positive feedback loop worth deliberately pursuing
     rather than under-investing and needing to revisit sooner.

5. **Recommend investing in current architecture quality as evolution prep**
   - Make the case that the single most effective way to prepare for future
     evolution is to keep the *current* architecture well-organized and
     well-documented, not to speculatively build in flexibility for
     evolution (see architecture-simplicity's caution against
     future-proofing).
   - A well-understood current architecture makes proposing, assessing, and
     executing the eventual evolution faster and more confident; a poorly
     understood or poorly documented one means that work has to happen
     under pressure, at the point evolution is actually needed.

6. **Recommend a regular, calendar-aligned review cycle**
   - Recommend a scheduled architectural review process rather than relying
     purely on ad hoc triggers — its purpose is twofold: a deliberate
     checkpoint to ask "is the current architecture still serving the
     system well," and a relief valve that defers speculative
     technology-adoption pressure ("should we be using X?") to a scheduled
     forum instead of relitigating it continuously.
   - Recommend timing the review to align with the organization's existing
     planning and budgeting calendar, so that if the architecture does need
     significant investment, that need surfaces while resources are still
     being allocated — and if the architecture is in good shape, that
     capacity can be confidently directed elsewhere.
   - Note that a review concluding "no changes needed" is a legitimate and
     valuable outcome, not a wasted cycle.

---

## AI decision guidance

When generating evolution-cadence guidance, keep these principles in mind:

- **Vision stability enables incremental progress; don't confuse the two
  levels:** frequent tactical delivery is healthy, frequent vision
  churn is not.
- **Novelty is not justification:** always ask for the concrete
  market/customer impact behind a technology-driven change proposal.
- **Switching costs include what you already know, not just what you'd
  gain:** make the sunk operational-expertise cost explicit before
  recommending a technology switch, especially in the "relevant but late"
  case.
- **Evolution is a when, not an if — plan its pacing, not its avoidance:**
  treat the goal as investing well when evolution happens, not preventing
  it from ever happening.
- **Prepare via current quality, not speculative flexibility:** a
  well-organized, well-documented architecture is the real hedge against
  a hard future evolution.
- **Give ad hoc pressure a scheduled outlet:** a regular review cycle
  absorbs "should we adopt X" energy that would otherwise destabilize
  day-to-day delivery.

---

## Success criteria

A strong evolution-cadence response should help teams achieve:

- **stable vision:** an architectural vision that changes rarely and
  deliberately, distinct from the steady cadence of incremental delivery
  toward it,
- **market-tested technology change:** technology-driven proposals held to
  a concrete customer/market-impact standard, not adopted on novelty alone,
- **honest switching-cost accounting:** sunk operational expertise weighed
  explicitly against a new technology's promised benefit before
  recommending a switch,
- **well-paced evolution:** evolution treated as inevitable but
  deliberately invested in when it happens, extending the time until the
  next evolution is needed,
- **a working review cadence:** a scheduled, calendar-aligned review
  process that both evaluates the architecture and absorbs ad hoc pressure
  to chase new technology outside of it.

---

## Example prompts for the AI

- "Our architectural vision seems to shift every quarter — help me figure
  out if that's actually a problem and what's driving it."
- "A new database technology is a better fit than what we're on, but we've
  been running our current one in production for years — how do I weigh
  the switch honestly?"
- "Help me set up a regular architecture review cadence that's actually
  useful and aligned with our planning calendar."

---

## Related guidance

Use this tool alongside:

- architecture-incremental-delivery
- architecture-capability-trajectory
- architecture-simplicity
- architecture-investment-mindset
