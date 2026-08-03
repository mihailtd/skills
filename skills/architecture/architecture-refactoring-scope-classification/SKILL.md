---
name: architecture-refactoring-scope-classification

description: Instructs the AI assistant to define refactoring precisely — restructuring existing code without changing its external behavior — and to classify every refactoring candidate as either Local (contained, reviewable in a single changeset) or Refactoring at Scale (a systemic problem spanning a substantial surface area, where no one can fully reason through a uniform change's effects the way they can for a single function) before deciding how much process, sequencing rigor, and rollout staging it actually needs.
---

# Refactoring Scope Classification Instructions

When supporting a decision about refactoring — whether a specific change
counts as one, and how much process it deserves — use this tool to ground
the assessment in a precise definition and an explicit scope classification,
rather than treating "refactor" as a vague catch-all for "clean up code."

---

## Purpose

This tool helps the AI assistant by:

- anchoring "refactoring" to its precise meaning — restructuring existing
  code without changing its external behavior — so a change that alters
  behavior isn't mislabeled as a refactor and given a lighter process than
  it actually needs,
- distinguishing a **Local** refactor (contained, reviewable in a single
  changeset) from a genuine **Refactoring at Scale** effort (a systemic
  problem spanning a substantial surface area), since the two need
  fundamentally different levels of rigor,
- explaining precisely what makes a refactor "at scale" — not a line-count
  threshold, but a qualitative shift in whether a person can still fully
  reason through a uniform change's effects across the whole affected area,
- connecting the at-scale case to the reality of refactoring a **live**
  system under a real deployment cadence, where the rollout/deployment
  strategy is part of the plan itself, not an afterthought.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- treats "restructuring without changing external behavior" as the
  non-negotiable definition of refactoring — a change that alters behavior,
  even subtly, isn't a refactor and shouldn't be planned, reviewed, or
  rolled out like one,
- classifies every refactoring candidate as Local or At-Scale explicitly,
  before deciding on process, and calibrates the rest of the effort
  (review depth, sequencing, rollout strategy) to that classification
  rather than applying one-size-fits-all rigor,
- reserves "Refactoring at Scale" for genuinely systemic problems — ones
  that require understanding one or more broad sections of the application
  and the discipline to propagate a fix strategically across the whole
  affected area, not just a large diff,
- treats deployment/rollout strategy as an integral part of an at-scale
  refactor's plan for any live, frequently-deployed system, since the
  difference between a quiet rollout and an outage is often exactly how
  the change was staged across deploys.

---

## Instructions for the AI

1. **Define refactoring precisely: restructuring without behavior change**
   - Treat any defined unit of code as a "system" — a set of code producing
     outputs from inputs, whether that's a single `if` statement, one
     function, or a whole application. Refactoring changes the system's
     internals; its observable behavior, from the outside, does not change.
   - Use this as a hard boundary, not a loose guideline: if a proposed
     change would alter what callers see (different outputs, new side
     effects, a different error path), it is not a refactor. It may still
     be a worthwhile change, but it needs the review, testing, and rollout
     process appropriate to a behavior change, not a refactor's lighter
     "prove nothing changed" process.

2. **Classify every candidate as Local or At-Scale before deciding on process**
   - **Local**: contained, with a natural boundary that fits in a single
     changeset a reviewer can comfortably evaluate. Most refactors are
     this — a function, a module, a small cluster of related files.
   - **Refactoring at Scale**: a refactor whose surface area is
     *substantial* — not defined by a fixed line-count threshold, but by a
     qualitative shift: at this size, no one can fully reason through the
     effect of a uniform change applied across the whole affected area the
     way they can for a single function or file.
   - Make the classification explicit for every candidate, and use it to
     calibrate everything downstream — a Local candidate doesn't need a
     formal change-proposal process or staged rollout; a genuine At-Scale
     candidate does.

3. **Recognize what actually makes a refactor "at scale"**
   - It requires (a) identifying a **systemic** problem, not a local one —
     the same pattern repeated across many places, not one messy function;
     (b) a solid understanding of one or more broad sections of the
     application, not just the immediate code being touched; and (c)
     enough discipline and stamina to propagate the fix strategically
     across the entire affected area, rather than opportunistically
     wherever it happens to be easiest.
   - Don't let "this touches a lot of files" alone qualify something as
     at-scale if the change in each file is trivial and mechanical (a
     pure rename, for instance, might touch hundreds of files but require
     none of the systemic reasoning that defines this category). The test
     is whether a person can still hold the full effect of the change in
     their head, not the raw diff size.

4. **Treat rollout strategy as part of an at-scale plan, not an afterthought**
   - An at-scale refactor usually intersects with refactoring a **live**
     system under a real, ongoing deployment cadence — old and new code
     coexist in production for real stretches of time, not an instant.
   - Build the deployment/rollout strategy (staging, canarying, feature-flag
     gating, order of service deploys) into the plan itself from the start.
     This is the concrete difference between refactoring a live,
     frequently-deployed system safely and causing an outage — treating it
     as a detail to "figure out later" is how an otherwise well-planned
     at-scale refactor still goes wrong at execution time.

---

## AI decision guidance

When generating refactoring-scope guidance, keep these principles in mind:

- **"Refactoring" means restructuring without behavior change, full stop**
  — a change that alters behavior needs a different process, regardless of
  how it's framed by whoever's proposing it.
- **Classify Local vs. At-Scale explicitly, before deciding on process** —
  don't let a candidate drift into heavier or lighter process than its
  actual scope warrants.
- **"At scale" is qualitative, not a line-count threshold** — the real
  test is whether someone can still fully reason through the change's
  effects across the whole affected area.
- **An at-scale refactor's rollout strategy is part of the plan, not a
  detail to defer** — this is where otherwise sound at-scale efforts most
  often go wrong in execution.

---

## Success criteria

A strong response should ensure that it:

- **holds the line on refactoring's definition** — no behavior change,
  regardless of how a proposed change is framed,
- **classifies every candidate as Local or At-Scale explicitly**, and
  calibrates review/process rigor to that classification,
- **applies the qualitative "at scale" test** — systemic problem, broad
  understanding required, strategic propagation — rather than a raw
  size/diff-count heuristic,
- **treats rollout strategy as integral to an at-scale plan**, not
  something to work out once the refactor itself is "done."

---

## Example prompts for the AI

- "Is renaming this variable across 200 files a refactor at scale, or is
  it just a big mechanical change?"
- "We want to restructure how this module handles errors — how much
  process does that actually need?"
- "Does this change behavior at all, or is it purely internal?"
- "How should we think about deploying a large-scale refactor without
  risking an outage?"

---

## Related guidance

Use this tool alongside:

- architecture-code-degradation-diagnosis — once a candidate is scoped,
  diagnosing *why* it degraded (requirement shift vs. tech debt) shapes
  what the refactor should actually turn it into.
- architecture-refactoring-decision-criteria — the triggers, when-not-to
  reasons, and risk checklist for deciding whether a scoped candidate is
  actually worth refactoring now.
- architecture-refactoring-plan-structure — how to structure the actual
  plan once a candidate is classified and justified, including staged
  rollout mechanics for At-Scale candidates specifically.
- architecture-urgent-vs-important — the general "scale the process to the
  actual need" principle this skill's Local-vs-At-Scale distinction is a
  specific application of.
- architecture-evolution-cadence — if a refactor amounts to an
  architectural-vision change rather than a local cleanup, treat it as the
  rare, deliberate event that skill describes.
