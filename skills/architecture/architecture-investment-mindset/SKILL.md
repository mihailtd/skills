---
name: architecture-investment-mindset
description: Instructs the AI assistant to evaluate proposed changes as investments with a return rather than as "short-term vs. long-term" tradeoffs, default-approve changes that fit within the current architecture (burden of objection on the objector), recast seemingly small changes in architectural terms to surface hidden cost, and frame architecture debates as options and trade-offs rather than as people or roles opposing each other.
---

# Investment Mindset Instructions

When supporting architecture design, use this tool to help teams evaluate
proposed changes the way an investor evaluates a bet — by its return, not by
a false short-term/long-term split — approve in-bounds changes by default,
surface the real architectural cost hiding inside "small" changes, and keep
the debate about options, never about people.

---

## Purpose

This tool helps the AI assistant by:

- reframing "short-term fix" language as a warning sign, not a reassurance —
  the maintenance, testing, and living-with-it costs of a change are long-term
  regardless of how the change was pitched,
- treating every proposed change as an investment decision with a return to
  evaluate, rather than forcing it into a short-term/long-term binary,
- establishing that changes realizable within the current architecture
  should be approved by default, with the burden on any objector to name a
  specific architectural flaw — not merely a stylistic preference,
- providing a concrete technique — recasting a "small" change in
  architectural terms — for surfacing costs (coupling, testing burden,
  dependability) that a change's surface-level size hides,
- insisting that architecture debates stay framed as options and trade-offs,
  never as one role or person's judgment pitted against another's.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- treats "let's just do the short-term fix" as a prompt to ask what the
  actual long-term cost of living with that fix will be, not as a free pass,
- evaluates every non-trivial change by asking "what's the return on this
  investment," with cost and benefit both made explicit,
- approves changes that fit inside the current architecture without
  friction by default, reserving real scrutiny for changes that don't, or
  for objections backed by a specific architectural concern,
- catches "small" changes that quietly introduce new relationships between
  previously unconnected components, and evaluates them by what that new
  relationship actually costs (testing surface, coupling, failure modes),
- keeps disagreements about a change framed in terms of the options and their
  trade-offs, actively steering away from framings that pit "architecture"
  against "engineering" or one person's judgment against another's.

---

## Instructions for the AI

1. **Treat "short-term" framing as a flag, not a discount**
   - When a change is pitched as a short-term fix, explicitly ask what its
     ongoing cost will actually be — maintenance burden, test coverage,
     technical debt interest — because those costs accrue over the long
     term regardless of the short-term label attached to the decision.
   - Avoid letting "short-term" function as an excuse to skip the same
     evaluation a "real" change would get; the label describes urgency, not
     exemption from consequences.

2. **Evaluate changes as investments, not as a short-term/long-term binary**
   - Reframe the question from "is this short-term or long-term" to "what
     return does this investment produce, and at what cost" — this applies
     uniformly whether the change is small or large, urgent or not.
   - Make the cost and the expected return explicit in the evaluation:
     what does the change cost to build and maintain, and what benefit
     (capability, risk reduction, velocity, revenue) does it produce in
     return.

3. **Default-approve changes that fit the current architecture**
   - When a proposed change is realizable within the system's existing
     architectural boundaries and relationships, treat it as sound by
     default — don't require it to justify itself from scratch.
   - Place the burden of objection on whoever wants to block it: a valid
     objection must point to a specific architectural flaw the change would
     introduce or worsen (e.g., a new tenuous invariant, broken component
     boundary, added coupling) — not general distaste, style preference, or
     "that's not how I'd do it."
   - Reserve deeper scrutiny for changes that actually require the
     architecture itself to evolve (see architecture-stages-of-change for
     scaling process rigor to scope).

4. **Recast "small" changes in architectural terms to find hidden cost**
   - When a change is described as small or trivial, actively test that
     framing by recasting it in architectural terms: does this introduce a
     relationship between components that didn't talk to each other before?
   - If it does, name the specific cost that new relationship introduces —
     e.g., "A never called B before; now it does — what does that cost in
     new test coverage, coupling between A and B's lifecycles, and A's
     dependability if B becomes unavailable?"
   - Use this technique specifically to catch changes that are small in
     lines-of-code but large in architectural consequence — the two are
     often uncorrelated, and surface-level size is not a reliable proxy for
     architectural cost.

5. **Frame debates as options and trade-offs, never as people versus people**
   - Actively steer change discussions toward comparing concrete options and
     their trade-offs, and away from framings that cast the debate as one
     role or person against another (e.g., "architecture vs. engineering,"
     "the architect says no").
   - When a debate starts drifting into people-versus-people framing,
     explicitly redirect it: restate the disagreement as two (or more)
     options with their respective costs and benefits, and evaluate those,
     not the people advocating for them.
   - Treat this as a facilitation responsibility, not an incidental nicety —
     framing debates around people rather than options damages working
     relationships and produces worse decisions, since it optimizes for
     "winning" the disagreement rather than for the best outcome.

---

## AI decision guidance

When generating change-evaluation guidance, keep these principles in mind:

- **"Short-term" is not a cost exemption:** always ask what the ongoing cost
  of a "short-term" decision will actually be.
- **Investment framing beats short/long-term framing:** evaluate cost versus
  return explicitly, for every change, regardless of its pitched urgency.
- **In-bounds changes are innocent until proven architecturally guilty:**
  don't require justification from scratch for changes that fit the current
  architecture; do require a specific architectural reason to block them.
- **Recast size in architectural terms before trusting it:** "small" is a
  claim about lines of code, not about architectural cost — check the
  latter directly using the new-relationship technique.
- **Always redirect people-vs-people framing to options-vs-options framing:**
  this is a facilitation move you should make proactively, not just when
  asked.

---

## Success criteria

A strong investment-mindset response should help teams achieve:

- **honest short-term accounting:** no "short-term fix" waved through
  without its real ongoing cost being named,
- **explicit investment evaluation:** cost and return articulated for
  changes under discussion, not left implicit,
- **low-friction in-bounds approval:** changes that fit the current
  architecture moving forward without needing to re-justify the whole
  architecture, while objections are held to a concrete-flaw standard,
- **surfaced hidden cost in "small" changes:** new component relationships
  introduced by ostensibly small changes named and evaluated explicitly,
- **options-based debate:** disagreements resolved by comparing trade-offs
  between concrete options, with no framing that pits people or roles
  against each other.

---

## Example prompts for the AI

- "We're calling this a short-term fix so we can ship it fast — help me
  figure out what it's actually going to cost us to live with long-term."
- "Someone's objecting to this change but I can't tell if it's a real
  architectural concern or just a style preference — how do I tell the
  difference?"
- "This change looks small on paper but now introduces a new dependency
  between two services — help me evaluate what that actually costs."
- "This debate is turning into 'architecture vs. engineering' — help me
  reframe it around the actual options on the table."

---

## Related guidance

Use this tool alongside:

- architecture-stages-of-change
- architecture-capability-trajectory
- architecture-simplicity
- architecture-incremental-delivery
- architecture-evolution-cadence
- architecture-alternative-generation
- architecture-urgent-vs-important
- architecture-risk-management
