---
name: architecture-refactoring-plan-structure

description: Instructs the AI assistant on structuring a refactoring execution plan — defining the end state with a Start/Goal/Observed metrics table (allowing both an ideal and a smaller acceptable end state), generating milestones via a three-question filter (attainable soon, valuable standalone, a safe pause point if interrupted), a repeatable-step-template pattern for refactors touching many similar structures, using order-agnostic milestones as a deliberate morale/parallelization lever, and closing every plan with a mandatory cleanup milestone (feature flags, shielding abstractions, dead code, stray comments, duplicated tests) without which a refactor is not considered done.
---

# Refactoring Plan Structure Instructions

When supporting the drafting of a refactoring execution plan — once a
candidate has been classified, diagnosed, and justified — use this tool to
structure the plan itself: the end state, the milestones, the sequencing,
and the mandatory close-out.

---

## Purpose

This tool helps the AI assistant by:

- requiring an explicit, measurable end state — a Start/Goal/Observed
  metrics table reusing the specific metric chosen for the candidate —
  rather than a prose description of the target that can't later be
  checked against,
- allowing both an *ideal* end state and a smaller *acceptable* one,
  since getting most of the way there often captures nearly all of the
  benefit,
- providing a concrete three-question filter for turning a raw list of
  steps into real milestones, so milestone boundaries are chosen
  deliberately rather than arbitrarily,
- recommending a repeatable-step-template approach for refactors that
  touch many similar structures, instead of a bespoke sequence invented
  per instance,
- treating order-agnostic milestones as a scheduling lever (for morale or
  parallelization) rather than an incidental detail,
- requiring a mandatory cleanup milestone in every plan — a refactor
  isn't done until its transitional artifacts are gone, not merely once
  the "real" work has landed.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- defines every refactoring plan's end state with a concrete metrics
  table (Metric | Start | Goal | Observed), reusing the specific metric
  chosen when the candidate was evidenced, and explicitly allows for an
  acceptable end state distinct from the ideal one,
- generates the milestone list by applying a three-question filter to a
  raw brainstormed step list, rather than inventing milestones
  unilaterally, and recommends generating that raw list *with* the people
  who'll execute the plan,
- applies a repeatable-step template — optionally behind a shielding
  abstraction built first — when a refactor's shape repeats across many
  similar structures, rather than re-deriving the sequence each time,
- uses order-agnostic milestones deliberately: to protect team morale on
  a long effort, or to parallelize genuinely independent work to finish
  sooner, stating explicitly which choice the plan is making,
- ends every plan with an explicit, mandatory cleanup milestone naming
  the specific transitional artifacts (feature flags, shielding
  abstractions, dead code, stray comments, duplicated tests) the
  candidate is expected to produce, tagged consistently so they can all
  be found in one search at close-out.

---

## Instructions for the AI

1. **Define the end state with a metrics table, not just prose**
   - Reuse the specific metric(s) already chosen to evidence the
     candidate (see **architecture-refactoring-evidence-gathering**) and
     lay out: `Metric | Start | Goal | Observed` — the `Observed` column
     left blank, to be filled in once the work actually lands, so success
     can be checked against the plan rather than asserted after the fact.
   - It's legitimate to define both an *ideal* end state and a smaller
     *acceptable* one — getting most of the way there often captures
     nearly all of the benefit. State this explicitly in the plan rather
     than implying "100% or nothing."

2. **Generate milestones with a three-question filter**
   - Take a raw, unordered list of candidate steps and filter each one
     through: (1) is it attainable within a reasonably short period — not
     so large it swallows a quarter, not so small it's noise; (2) does
     reaching it deliver real, standalone value on its own (to other
     engineers, not just to the refactor's own progress) — prefer steps
     that demonstrate the refactor's benefit early and often over ones
     that only pay off at the very end; (3) if work had to stop right
     after this step (a shifted priority, an incident, a reorg — treat
     this as a realistic possibility, not a hypothetical), would the
     codebase be left in a safe, coherent state rather than a confusing
     half-done one?
   - A step that fails the third question risks the incomplete-refactor
     risk in miniature — don't propose a milestone boundary that lands
     there.
   - Recommend generating the raw candidate-step list itself *with the
     team* — a solo timeboxed brainstorm revisited a day later, or a
     group session (sticky notes or equivalent) merged and pruned
     together — rather than inventing the full step sequence
     unilaterally. Supply the structure and the filter; let the people
     who'll execute the plan generate and prune the actual steps.

3. **Use a repeatable-step template for refactors touching many similar structures**
   - For a candidate that repeats across many similar structures (many
     services, many repos, many modules needing the same
     transformation), recommend a single repeatable step template applied
     one instance at a time, rather than a bespoke sequence per instance.
   - This keeps each application small, lets the team coordinate with
     whichever owners are affected one at a time, and keeps the amount
     of the codebase "in flux" at any moment as small as possible.
   - If it's worth the investment, recommend building a shielding
     abstraction *first*, so the transition detail is hidden behind one
     seam rather than exposed at every call site while the migration is
     in progress.

4. **Use order-agnostic milestones as a deliberate scheduling lever**
   - Once real prerequisites are mapped, note which remaining milestones
     have no strict ordering constraint between them, and use that
     flexibility deliberately: insert an easier, standalone milestone
     between two difficult ones to protect team morale on a long effort,
     or parallelize genuinely independent milestones to finish sooner.
   - State which choice the plan is making and why, rather than leaving
     the ordering arbitrary.

5. **End every plan with a mandatory cleanup milestone**
   - A refactor is not done until this step is done — enumerate the
     specific transitional artifacts this candidate is expected to
     produce and require their removal as the plan's last step, not an
     optional follow-up:
     - **Feature flags** used to gate the rollout — remove promptly once
       fully enabled, not left indefinitely. This is a real, measurable
       cost, not just tidiness: stale, fully-enabled feature flags left
       in place have been measured consuming a non-trivial share (close
       to 5% in one documented case) of average backend request
       execution time before cleanup.
     - **Shielding abstractions** built to hide the transition — once the
       cutover is complete and stable, simplify or remove them so they
       don't look like a still-in-progress migration to a future reader.
     - **Dead code** — the old implementation, once fully replaced.
     - **Comments** — stray TODOs, "code in flux" warnings, notes about
       pending removal — delete them; commented-out or stale-transition
       comments should never be left in place once their purpose has
       been served.
     - **Duplicated tests** written to verify the new implementation
       alongside the old one's existing suite — consolidate once the old
       path is gone, both for clarity and for test-suite speed.
   - Recommend a single, consistent, greppable tag for every transitional
     artifact the plan introduces (e.g. `TODO: <candidate-name> cleanup`)
     so the final cleanup pass can find all of them in one search rather
     than relying on memory.

---

## AI decision guidance

When generating refactoring-plan guidance, keep these principles in mind:

- **Define success with a table, not prose** — Start/Goal/Observed, with
  both an ideal and acceptable end state where relevant.
- **Filter a raw step list into milestones with the three-question test**
  — attainable soon, valuable standalone, safe to pause after.
- **Generate the raw step list with the team**, not unilaterally.
- **Repeatable transformations get one template, applied instance by
  instance** — not a bespoke plan per instance.
- **Order-agnostic milestones are a lever to use deliberately** — for
  morale or for parallelization — not an incidental fact.
- **No plan is complete without an explicit cleanup milestone** — a
  refactor isn't done until its transitional artifacts are gone.

---

## Success criteria

A strong response should ensure that it:

- **includes a concrete Start/Goal/Observed metrics table** for the
  candidate's end state, with an acceptable-end-state option where
  relevant,
- **derives milestones via the three-question filter**, and recommends
  generating the raw step list collaboratively,
- **applies a repeatable-step template** where a refactor's shape repeats
  across many similar structures,
- **uses order-agnostic sequencing deliberately**, stating the reasoning,
- **ends with a named, mandatory cleanup milestone** enumerating the
  specific transitional artifacts expected, tagged for easy discovery.

---

## Example prompts for the AI

- "How should we define 'done' for this refactor?"
- "We have a huge list of steps for this migration — how do we turn that
  into real milestones?"
- "We need to do the same migration across 40 services — how should we
  structure that?"
- "Is this refactor actually finished, or is there cleanup we're missing?"

---

## Related guidance

Use this tool alongside:

- architecture-refactoring-evidence-gathering — the source of the
  specific metric that becomes this plan's Start/Goal/Observed row.
- architecture-refactoring-decision-criteria — the incomplete-refactor
  risk this skill's mandatory cleanup milestone and three-question
  milestone filter both directly address.
- architecture-verified-behavior-rollout — the staged-rollout technique
  for an At-Scale candidate that swaps an implementation and needs
  behavioral-equivalence verification; its dispatcher/shielding
  abstraction is exactly the kind of artifact this skill's cleanup
  milestone requires removing once stable.
- architecture-incremental-delivery / architecture-incremental-design —
  the general incremental-delivery discipline this skill's milestone
  structuring is a specific application of.
- architecture-change-proposals / architecture-change-templates —
  capturing the plan itself as a formal change proposal, when the scope
  warrants that level of process.
