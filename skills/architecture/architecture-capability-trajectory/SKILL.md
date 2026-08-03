---
name: architecture-capability-trajectory
description: Instructs the AI assistant to size architectural investment in a capability by its trajectory — rate of change crossed with uncertainty/scope of future change — rather than by its next known requirement, drawing on both product-driven and technology-driven signals and evaluating new technology soberly against real switching costs.
---

# Capability Trajectory Instructions

When supporting architecture design, use this tool to help teams decide how
much architectural investment a capability deserves by projecting its
trajectory — how fast it's likely to change and how much its future scope is
still unknown — instead of sizing the investment to whatever the next
requirement happens to ask for.

---

## Purpose

This tool helps the AI assistant by:

- separating a capability's *next requirement* from its *trajectory*, and
  insisting the trajectory is what should drive the architectural investment
  decision,
- applying a two-axis model — rate of change crossed with uncertainty/scope
  of future change — to classify a capability into a quadrant that implies a
  concrete investment posture,
- treating product management's forward intent (not just current backlog
  items) as a first-class input to architectural decisions, and prompting
  the AI to actively elicit it when it's missing,
- distinguishing product-driven change from technology-driven change as the
  two sources that shape a capability's trajectory, and reasoning about
  when they align versus conflict,
- evaluating new technology adoption soberly — weighing genuine capability
  or efficiency gains against real switching costs, rather than adopting or
  dismissing it on hype or reflexive skepticism alone.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- can place any capability under discussion on the rate-of-change /
  uncertainty grid and explain what that placement implies for how much
  architectural flexibility to build in now,
- avoids both failure modes: under-investing in a capability trending toward
  high change and high uncertainty, and over-investing (speculative
  generality) in a capability that's genuinely a one-time effort,
- asks product management directly about a capability's expected trajectory
  when it isn't already known, rather than inferring it silently from the
  current ticket,
- separately tracks technology-driven pressure to change (new platforms,
  tools, or paradigms) and reasons about whether it reinforces or fights the
  product-driven trajectory,
- evaluates a candidate new technology on real merits — measurable
  capability or efficiency gain, likely adoption success, and switching cost
  — before recommending its adoption, and stays skeptical of hype-driven
  pushes to adopt.

---

## Instructions for the AI

1. **Distinguish trajectory from next requirement**
   - When a change or new capability is proposed, explicitly ask (or infer
     from context and flag the inference) not just "what does this
     requirement need" but "where is this capability headed over its
     lifetime."
   - Make clear that architecture should be sized to trajectory, not to the
     literal scope of the next known requirement — under-building for a
     fast-moving, uncertain capability creates near-term rework; over-building
     for a stable, well-understood one wastes effort and adds needless
     complexity.

2. **Apply the two-axis trajectory model**
   - **Axis 1 — rate of change:** how often is this capability expected to
     change?
   - **Axis 2 — uncertainty / scope of future change:** how well understood
     is the *nature* of future changes, and how large might they be?
   - Reason explicitly about which quadrant a capability falls into, and
     state the implied posture:
     - **Low rate, low uncertainty (one-time effort):** build the simplest
       thing that satisfies the known requirement; do not add flexibility
       for change that isn't coming (e.g., a one-off "save as PDF" feature
       with no roadmap for further export formats).
     - **High rate, low uncertainty (evolve the design):** invest in a
       design that's easy to extend along the known dimension of change,
       without necessarily restructuring the architecture.
     - **Low rate, high uncertainty:** favor deferring commitment — avoid
       hard-wiring assumptions that a rare-but-unpredictable future change
       would force you to unwind.
     - **High rate, high uncertainty (evolve the architecture):** this is
       where real architectural investment pays off — build in seams,
       abstraction boundaries, and configurability precisely because both
       the frequency and shape of future change are hard to predict.
   - Treat the quadrant as a starting posture to reason from, not a rigid
     formula — call out when a specific capability's real-world context
     should shift it toward more or less investment than the quadrant alone
     implies.

3. **Elicit trajectory from product management, don't assume it**
   - When trajectory isn't already stated, prompt for it directly: what's
     the product roadmap for this capability, is more investment planned in
     this area, is this a strategic differentiator or a checkbox feature?
   - Treat this as a genuine cross-functional input to the architecture
     decision, not a one-time kickoff question — trajectories change as
     product strategy evolves, so revisit the assumption when a capability's
     context shifts materially.

4. **Separate technology-driven change from product-driven change**
   - Recognize that a capability's trajectory is shaped by two distinct
     forces: what the product needs to do (product-driven), and what the
     underlying technology landscape makes newly possible, necessary, or
     obsolete (technology-driven).
   - Reason about whether the two are aligned (a product need and a
     technology shift point the same direction, reinforcing the case for
     investment) or in conflict (the technology world is moving one way
     while the product doesn't need it yet, or vice versa) — and be explicit
     about which situation is in play.
   - Recommend focusing technology-trajectory monitoring effort where it
     matters most: areas of rapid technological change, and areas where the
     current technology is genuinely underserving the system's needs.
     Don't spread trajectory-tracking effort evenly across stable,
     well-served technology choices.

5. **Evaluate new technology adoption soberly**
   - When technology-driven change is proposed (adopting a new tool,
     platform, or paradigm), assess it on its actual merits: what capability
     or efficiency gain does it offer, how likely is it to succeed and
     mature versus stall or get abandoned, and what does switching actually
     cost (migration effort, retraining, ecosystem rebuilding)?
   - Push back on adoption driven mainly by hype or novelty when the
     switching cost isn't justified by a correspondingly real gain.
   - Recognize the flip side too: new technology can be worth adopting for
     reasons beyond raw capability, e.g., recruiting/retaining talent who
     want to work with current tools, or unlocking efficiency gains that
     compound over the capability's expected lifetime — factor these in
     rather than requiring the gain to be purely functional.

---

## AI decision guidance

When generating trajectory-based investment guidance, keep these principles
in mind:

- **Trajectory, not ticket, sizes the investment:** always reason one level
  above the literal requirement in front of you.
- **Both axes matter independently:** a high rate of change with low
  uncertainty needs a different answer than high uncertainty with a low
  rate of change — don't collapse the model to a single "how much will this
  change" question.
- **Ask, don't assume, when trajectory is unclear:** treat missing
  trajectory information as a gap to close with product management, not
  something to guess silently.
- **Alignment vs. conflict between drivers is itself useful signal:** call
  out explicitly when technology-driven and product-driven pressures point
  the same way (stronger case) or different ways (requires explicit
  tradeoff).
- **New technology needs a real cost/benefit case:** neither reflexive
  adoption nor reflexive skepticism is a substitute for weighing actual gain
  against actual switching cost.

---

## Success criteria

A strong trajectory-based response should help teams achieve:

- **explicit quadrant placement:** every non-trivial capability under
  discussion mapped to a rate-of-change / uncertainty quadrant with a stated
  rationale,
- **investment matched to trajectory:** architectural effort that visibly
  scales up for high-rate/high-uncertainty capabilities and scales down for
  one-time-effort capabilities,
- **elicited product intent:** trajectory informed by an actual product
  management input, not an unstated assumption,
- **separated change drivers:** product-driven and technology-driven
  pressures reasoned about individually, with their alignment or conflict
  made explicit,
- **justified technology adoption:** any recommended new technology backed
  by a specific capability/efficiency gain that outweighs its stated
  switching cost.

---

## Example prompts for the AI

- "We're building a one-off export feature — should we architect it for
  future extensibility, or just build the simplest thing that works?"
- "This capability seems to be changing constantly and we don't know why —
  help me figure out if that's product-driven or technology-driven."
- "The team wants to adopt a new framework because it's trending — help me
  evaluate whether the switching cost is actually justified."

---

## Related guidance

Use this tool alongside:

- architecture-stages-of-change
- architecture-simplicity
- architecture-investment-mindset
- architecture-standards-adoption
- architecture-evolution-cadence
- architecture-incremental-delivery
