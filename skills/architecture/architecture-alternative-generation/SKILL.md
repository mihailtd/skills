---
name: architecture-alternative-generation
description: Instructs the AI assistant to generate and compare multiple conceptual alternatives for nontrivial changes as discrete, separately-evaluated proposals — countering anchoring on the first idea, capping alternatives to a manageable number, and treating rejection of an alternative (or of all of them) as a healthy, valuable outcome rather than a failure.
---

# Alternative Generation Instructions

When supporting architecture design, use this tool to help teams generate
real alternatives at the conceptual stage of a change, evaluate them as
discrete proposals rather than folding them into one, avoid anchoring on
whichever idea arrived first, and treat rejecting an alternative — or every
alternative — as evidence the process is working, not a sign of failure.

---

## Purpose

This tool helps the AI assistant by:

- treating the conceptual stage as the best opportunity to generate and
  compare genuinely different approaches to the same motivation, rather
  than letting the first workable idea proceed unexamined,
- insisting that each alternative be captured as its own discrete change
  proposal rather than embedded as options within a single proposal — a
  proposal offering "either A or B" is just a restatement of the
  motivation, not a decision aid,
- explaining the cost of anchoring on a single early idea: issues
  discovered late are expensive to fix, and by then cognitive investment
  and sunk-cost bias make a fair comparison with alternatives difficult,
- providing concrete techniques for provoking alternatives when a team
  isn't naturally generating them, and calibrating how many alternatives
  are actually worth carrying (a handful, not many),
- reframing rejection — of one alternative, or of every alternative
  considered — as a valuable, healthy outcome of a working process, since
  the earlier a bad idea is caught, the less is wasted on it.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- actively generates more than one conceptual approach for nontrivial
  changes, using the conceptual stage specifically for this purpose,
- keeps each alternative as its own proposal, evaluated on its own costs and
  characteristics, with relationships between alternatives tracked (e.g.,
  linked to a shared motivation) rather than merged into one document,
- surfaces issues with a leading approach early, while alternatives are
  still cheap to develop and compare, instead of discovering them late when
  switching costs and attachment bias make a fair comparison hard,
- uses lightweight techniques (including intentionally weak suggestions as
  provocation) to get a stalled team generating real alternatives, while
  capping the count to a manageable number and keeping each proposal
  lightweight enough that generating it isn't itself a burden,
- treats a rejected alternative, or a fully rejected proposal set, as a
  legitimate, valuable result — and identifies specifically what clarity
  the process forced (problem definition, implementation cost, operational
  cost) that led to the rejection.

---

## Instructions for the AI

1. **Use the conceptual stage to generate real alternatives**
   - For any nontrivial or ambiguous change, prompt for more than one
     conceptual approach to the same motivation before moving toward
     design — a single-alternative outcome is fine for small, incremental
     changes where the current architecture points clearly to one path, but
     shouldn't be assumed for larger or more open-ended changes.
   - Keep each alternative's motivation the same and let the conceptual
     approach differ — this is what makes a genuine comparison possible
     (same problem, different proposed solutions with different costs and
     characteristics).

2. **Never embed alternatives inside a single proposal**
   - If a proposal reads as "we could do A or B," recognize this as a
     restated motivation, not a decision aid, and split it into two
     discrete change proposals.
   - Recommend keeping these separate proposals linked to each other and to
     the shared motivation/requirements in the backlog, so decision-makers
     can see the full set of considered alternatives at once.
   - Structuring the debate this way also leaves a clean record of what
     alternatives were considered and why one was chosen over the others —
     useful when the same tradeoff resurfaces later.

3. **Counter anchoring on the first idea**
   - Explain the concrete cost of skipping alternative-generation: when a
     team proceeds with its first idea and issues surface late, corrections
     become expensive, sometimes requiring an entirely different conceptual
     approach after significant investment is already sunk.
   - Name the bias explicitly when it's in play: participants naturally
     grow attached to an approach they've already invested time in, and
     freshly-sketched alternatives can look artificially strong by
     comparison (the leading approach also looked great before its issues
     surfaced) — both effects make a late comparison less trustworthy than
     an early one.
   - Recommend front-loading alternative exploration specifically to avoid
     this dynamic — the cost of comparing alternatives early is far lower
     than the cost of discovering a fatal issue late.

4. **Use concrete techniques to provoke alternatives when they aren't emerging**
   - If a team defaults to a single approach without real debate, ask
     directly for alternatives; if that doesn't produce anything, propose
     some yourself — they don't need to be good, and a deliberately weak
     suggestion can be an effective way to prompt someone else to counter
     with a better idea.
   - Expect early-stage alternatives to often be variations or combinations
     of each other rather than fully distinct approaches — this is normal
     and still useful raw material for comparison.

5. **Calibrate the number of alternatives**
   - Recommend keeping the number of alternatives considered for a single
     motivation modest — roughly four or fewer, even for a major change;
     more than that tends to create its own overhead without adding
     proportional insight.
   - Keep each conceptual proposal lightweight (a short document, not a
     full design) so generating and comparing several isn't itself a
     burdensome undertaking.
   - When multiple alternatives remain genuinely competitive, it's
     reasonable to advance more than one into detailed design in parallel —
     but minimize how many are carried this far, since detailed design work
     is real effort that should be spent deliberately, not as a substitute
     for making a decision.

6. **Look for consolidation opportunities across alternatives or requirements**
   - When evaluating a set of alternatives, also check whether a single
     proposal could address a broader set of needs than the one motivation
     it started from — sometimes several related requirements point to the
     same underlying mechanism, and identifying that is a bigger win than
     picking the best alternative for a narrower motivation.

7. **Don't let the conceptual stage drag on**
   - When a change is straightforward and no real alternatives exist,
     recommend moving ahead quickly rather than manufacturing debate.
   - When there are genuine, complex alternatives to weigh, allow more time,
     but don't let the decision drag on unreasonably — the point of
     generating alternatives is a better decision, not an open-ended
     deliberation.

8. **Treat rejection — partial or total — as a healthy, valuable outcome**
   - Make explicit that a strong architectural practice will produce many
     conceptual proposals and reject a substantial share of them — this is
     a sign of two good things happening: the team is genuinely creative
     (producing a real variety of approaches, not just refining the first
     one), and the process is drawing in broad-based contributions rather
     than only the ideas of the most dominant or prolific contributors.
   - Frame the winnowing-out process as collaborative refinement, not
     competition — the process of assessing even the proposals that get
     rejected typically strengthens whichever one is ultimately approved.
   - Recognize that a fully rejected proposal set is also a legitimate,
     valuable outcome. A strong change process forces clarity on: whether
     the problem itself was correctly understood, whether the
     implementation cost is justified by the return (see
     architecture-investment-mindset), and whether ongoing operational cost
     (compute, storage, hardware) makes the change worthwhile at all. When
     that clarity leads to rejecting every proposal, that's the process
     successfully avoiding an expensive dead end — the earlier this happens
     in the stages (motivation/concept, rather than detailed design,
     implementation, or after release), the more waste is avoided.

---

## AI decision guidance

When generating alternative-generation guidance, keep these principles in
mind:

- **One idea is not a decision — insist on real comparison for nontrivial
  changes:** don't let the first workable approach proceed unexamined when
  the change is significant enough to warrant options.
- **Alternatives are separate proposals, not options within one:** treat
  "A or B" framing as an unfinished proposal-splitting task.
- **Early comparison beats late discovery, structurally:** name anchoring
  and sunk-cost bias explicitly when a team is reluctant to compare
  alternatives to an already-invested-in approach.
- **A handful of lightweight alternatives, not many heavy ones:** keep the
  count around four or fewer and each proposal cheap to produce.
- **Rejection is success signal, not failure signal:** always reframe a
  rejected alternative or a fully rejected proposal set as the process
  working, and name specifically what clarity it forced.

---

## Success criteria

A strong alternative-generation response should help teams achieve:

- **genuine alternatives generated:** more than one conceptual approach
  considered for nontrivial changes, sharing a motivation but differing in
  approach,
- **discrete, linked proposals:** each alternative captured as its own
  proposal, cross-referenced to its shared motivation and to competing
  alternatives,
- **early issue discovery:** problems with a leading approach surfaced
  while alternatives are still cheap to compare, not after heavy investment,
- **calibrated effort:** a manageable number of lightweight alternatives
  rather than either a single unexamined idea or an unwieldy long list,
- **rejection treated as healthy:** rejected alternatives and fully rejected
  proposal sets explicitly named as valuable outcomes, with the specific
  clarity gained (problem, implementation cost, operational cost) called
  out.

---

## Example prompts for the AI

- "We only have one idea for how to solve this — help me figure out whether
  we should be generating alternatives before committing to a design."
- "This proposal says 'we could either do X or Y' — help me split that into
  separate, comparable proposals."
- "We just spent a lot of effort on a proposal and are now finding out it
  won't work — how do I stop the team from feeling like this was wasted
  effort?"
- "We evaluated three approaches to this problem and rejected all of them —
  is that a bad outcome?"

---

## Related guidance

Use this tool alongside:

- architecture-change-proposals
- architecture-stages-of-change
- architecture-investment-mindset
- architecture-urgent-vs-important
