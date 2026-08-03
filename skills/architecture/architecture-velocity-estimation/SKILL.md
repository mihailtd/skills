---
name: architecture-velocity-estimation
description: Instructs the AI assistant to treat schedules and budgets as real design inputs rather than arbitrary constraints — estimating with historical, bucketed ranges instead of false precision, tracking detailed-design timing specifically to build a predictive baseline, using deviation from that baseline as an early warning signal, and rejecting architectural approaches that don't fit the time actually available.
---

# Velocity Estimation Instructions

When supporting architecture design, use this tool to help teams treat
schedule and budget as legitimate architectural design inputs — estimating
with honest, historically-grounded ranges instead of false precision,
tracking the detailed-design stage specifically to build a predictive
baseline, using deviation from that baseline as an early warning sign, and
rejecting a conceptual approach that doesn't actually fit the time
available rather than pretending it will.

---

## Purpose

This tool helps the AI assistant by:

- reframing schedules, budgets, and available people as legitimate inputs
  to architectural design, not arbitrary external constraints to work
  around or resent,
- redefining the design goal as finding an architecture that meets
  requirements *and* fits the project's actual parameters — not finding
  the abstractly "best" architecture, a label that's meaningless without
  defined criteria,
- warning against false precision in estimation — the difference between
  "30 days" and "31 days" is meaningless, while the difference between "a
  few days" and "a few months" is not,
- recommending range-based estimates built from historical, bucketed data
  (small/medium/large) rather than single-point guesses, since a single
  number falsely anchors expectations and hides real uncertainty,
- recommending tracking specifically the detailed-design stage's duration
  (not the more variable motivation/concept stages) to build a genuinely
  useful predictive dataset, and using deviation from that "typical" range
  as an early, actionable warning sign,
- insisting that when a conceptual approach doesn't fit the time actually
  available, the right response is to look for a different approach, not to
  proceed anyway on an unrealistic timeline.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- incorporates architecture cost, implementation cost, and operational cost
  into the design process itself, rather than treating schedule/budget as
  someone else's problem to negotiate separately,
- estimates using ranges grounded in the team's own historical data,
  bucketed by rough change size, rather than single-point guesses or
  unfounded new estimates for each change,
- tracks the detailed-design stage's start, review start, and approval
  dates specifically, since this stage's duration is the most useful signal
  (motivation/concept timing varies too much to be predictive),
- treats a design that's running well past its expected "typical" range as
  an actionable red flag worth investigating, rather than something to
  quietly push through,
- rejects — or actively seeks an alternative to — a conceptual approach
  whose required time clearly exceeds what's actually available, rather
  than committing to an approach the team already knows won't fit.

---

## Instructions for the AI

1. **Treat schedule, budget, and people as design inputs**
   - Recommend explicitly incorporating architecture cost, implementation
     cost, and operational cost into the design process, rather than
     designing first and discovering feasibility problems later.
   - Reframe the design goal away from finding the "best" architecture (an
     undefined, unmeasurable target) and toward finding an architecture
     that meets the requirements and that the team can actually build given
     real project parameters.

2. **Avoid false precision; use ranges**
   - Push back on unnecessarily precise estimates (e.g., "31 days") — what
     actually matters early on is knowing whether a change is a matter of
     days, weeks, or months.
   - Recommend expressing estimates as ranges (e.g., "4 to 6 weeks") rather
     than single numbers — a single number anchors expectations around one
     point and hides the real uncertainty, while a range forces that
     uncertainty to be visible and communicates its size directly.

3. **Build and use a historical, bucketed dataset**
   - Recommend recording, for past changes, when detailed design started,
     when review started, and when approval was reached — this data is
     what makes future estimation fast and grounded rather than
     speculative.
   - Recommend bucketing past changes into a small number of size
     categories (e.g., small/medium/large) when timing data shows wide
     variance, so a new change can be compared against similarly-sized past
     work rather than the dataset as a whole.
   - When starting a new change, recommend a brief (roughly five-minute)
     lookup against this data — "which past changes were like this one, and
     what range did they take?" — instead of an uninformed, from-scratch
     guess.
   - Deliberately don't bother tracking motivation/concept-stage timing —
     it varies too widely to be a useful predictive signal; focus tracking
     effort on the detailed-design stage specifically.

4. **Use deviation from the "typical" range as an early warning signal**
   - Once a "typical" range is established (e.g., 4-6 weeks for a medium
     detailed design), treat a design that's clearly exceeding that range
     as an actionable signal to investigate — not proof of a specific
     problem, but a reliable prompt to find out what's going on (a
     higher-priority interruption, an underestimated scope, unsettled
     requirements, or something else).
   - Recognize the added benefit of having an established range at all:
     knowing the expected timeline focuses the author's effort from the
     start ("finish this within the known typical window," not "finish
     this whenever"), which tends to make the detailed-design stage more
     predictable over time — a feedback loop worth deliberately
     establishing.

5. **Reject approaches that don't fit the time available**
   - When a proposed conceptual approach would clearly take longer than the
     time actually available (e.g., three months of detailed design against
     one month available), recommend looking for a different approach
     rather than proceeding on an unrealistic timeline.
   - Acknowledge this isn't always possible, but treat a more
     expensive/slower approach as requiring strong, explicit justification
     — there's nothing effective about an architecture practice that
     defaults to the slower, costlier path when a workable faster
     alternative exists (see architecture-alternative-generation).

---

## AI decision guidance

When generating velocity-estimation guidance, keep these principles in
mind:

- **Schedule and budget are architecture inputs, not obstacles to the
  architecture:** always fold them into the design conversation itself.
- **Precision should match actual knowledge:** don't recommend
  single-point estimates when the underlying uncertainty is real — use
  ranges sized to reflect it.
- **History beats guessing, every time:** always check for relevant
  historical data (bucketed by size) before producing a fresh estimate.
- **Track the stage that's actually predictive:** detailed design, not
  motivation/concept, is where timing data is worth collecting.
- **A blown range is a prompt to investigate, not to ignore:** treat
  deviation from the established typical window as actionable signal.
- **Fit the approach to the time, not the time to the approach:** when they
  don't match, the default recommendation is to change the approach.

---

## Success criteria

A strong velocity-estimation response should help teams achieve:

- **schedule-aware design:** architecture, implementation, and operational
  cost considered as part of the design process itself,
- **honest ranges:** estimates expressed with a range that reflects real
  uncertainty, not false single-point precision,
- **a working historical baseline:** detailed-design timing tracked and
  bucketed by size, usable for fast, grounded estimation on new changes,
- **early drift detection:** deviations from the typical range caught and
  investigated promptly, rather than discovered only at a missed deadline,
- **feasible approach selection:** a conceptual approach chosen (or
  reconsidered) based on genuine fit with the time actually available.

---

## Example prompts for the AI

- "We need to estimate how long this change will take — help me use our
  historical data instead of guessing from scratch."
- "This detailed design has been going for eight weeks when our typical
  range is four to six — what should we do about it?"
- "Our preferred architectural approach would take three months but we only
  have one — how should we think about this?"

---

## Related guidance

Use this tool alongside:

- architecture-backlog-management
- architecture-review-process
- architecture-alternative-generation
- architecture-investment-mindset
