---
name: code-review-naming-consistency
description: Guides AI reviewers to enforce naming consistency using "name molds" (the structural pattern combining concepts in a name, e.g. max_X_per_Y vs X_max_num) and Feitelson's three-step naming model (select concepts, choose words per concept with a project lexicon, then construct the name using a consistent mold) — directly targeting teams where the same idea gets named a different way every time.
---

# Code Review: Naming Consistency

This skill helps AI reviewers diagnose and fix the specific failure mode
where a codebase expresses the same underlying concept using many different
name shapes — e.g., `max_benefit`, `benefit_max_num`, `max_monthly_benefit`
all meaning the same thing. It introduces "name molds" as the concrete unit
of consistency to check for, and Feitelson's validated three-step model as
the concrete procedure for constructing a new name that fits.

---

## When to use this skill

Use this skill when you need to:

- diagnose why a codebase feels inconsistent even though individual names
  look reasonable in isolation,
- decide how to name something new so it matches how similar things are
  already named elsewhere in the codebase,
- resolve disagreement between team members proposing different names for
  the same concept,
- set up naming conventions for a new project or module before inconsistent
  patterns take hold,
- run a naming-consistency audit across an existing codebase or module.

---

## Outcome

Produce a naming-consistency review that:

- identifies the "mold" (structural pattern) a given name follows, and
  checks it against the mold(s) already established for similar concepts in
  the codebase,
- explains why mold inconsistency is a real comprehension cost, not just an
  aesthetic complaint — it forces extra searching effort to locate the core
  concept in a name, and it prevents a reader's memory from pattern-matching
  across genuinely related code,
- recommends converging on a small, deliberately limited number of molds
  per codebase, rather than tolerating however many molds happen to have
  emerged organically,
- applies Feitelson's three-step model (select concepts → choose words per
  concept → construct the name using a consistent mold) as the concrete
  procedure for producing a new name, since this model has been
  experimentally shown to produce measurably better-rated names than
  unguided naming,
- recommends a project lexicon to resolve synonym ambiguity (e.g., is it
  "delete" or "remove"? "customer" or "client"?) before it produces
  inconsistent word choices across the codebase.

---

## Instructions for the AI

1. **Identify the "mold" a name follows, not just its individual words**
   - A name mold is the structural pattern combining a name's parts,
     independent of the specific words used — e.g., `max_X`, `max_X_per_Y`,
     `X_max_num`, and `max_Y_X` are four different molds that could all
     express "the maximum benefit per month."
   - When reviewing a new or changed name, explicitly identify which mold it
     uses, and check whether that mold matches the mold already used for
     structurally similar concepts elsewhere in the codebase (e.g., other
     "maximum X per Y" values, other "list of X" values, other "X
     converted to Y" values).
   - Treat a name that uses a *different* mold from its established peers as
     a real inconsistency, even when the underlying words are otherwise
     reasonable — e.g., flag `interest_maximum` if the codebase's
     established mold for this concept is `max_X` (as in `max_benefit`),
     even though "interest_maximum" is a perfectly clear name in isolation.

2. **Explain why mold inconsistency has a real cognitive cost**
   - When multiple molds coexist for the same kind of concept, a reader has
     to search for the core concept in a variable-position location within
     each name, adding extraneous effort that isn't spent on actually
     understanding the code.
   - Mold inconsistency also weakens a reader's ability to use prior
     experience: a name like `max_interest_amount` is more likely to remind
     a reader of previously-seen code like `max_benefit_amount` (same mold)
     than of `interest_maximum` (different mold) — even when the
     underlying logic is otherwise similar. Frame consistency feedback in
     these concrete recall/pattern-matching terms.

3. **Recommend converging on a small, deliberate set of molds**
   - For an existing codebase, recommend extracting a list of currently-used
     names for a relevant scope (a module, a feature, a class) and
     identifying which molds are already in use — then picking a small
     number of them (ideally one per structural pattern, e.g., one mold for
     "maximum of X," one for "X filtered by Y") as the go-forward standard.
   - For a new project or module, recommend agreeing on the molds to use
     before much naming has happened — since (see
     code-review-naming-fundamentals) initial naming decisions tend to
     persist, this is a high-leverage moment to get right.
   - When flagging an inconsistent name during review, recommend the fix in
     terms of the established mold specifically (e.g., "use the `max_X`
     mold used elsewhere in this module") rather than just proposing an
     alternative name in isolation.

4. **Apply Feitelson's three-step model to construct new names**
   - **Step 1 — select the concepts to include.** Identify what
     information the name actually needs to convey based on its intent —
     what the object holds and what it's used for. If a comment would
     otherwise be needed to clarify the name (e.g., "this is in
     kilometers," "this buffer holds unvalidated input"), that's a signal
     the concept should be part of the name itself. Consider whether a
     value changing meaning at some point (e.g., raw input becoming
     validated) warrants a distinct name for the new state, rather than
     reusing the same name across a meaningful transition.
   - **Step 2 — choose the words for each concept.** Prefer the word
     already established in the domain or the codebase over introducing a
     new synonym. When multiple reasonable word choices exist, resolve the
     ambiguity deliberately rather than letting different authors pick
     different synonyms — see step 5 on maintaining a project lexicon.
   - **Step 3 — construct the name using a consistent mold.** Apply the
     mold already established for structurally similar names in the
     codebase (see steps 1-3 above). When no mold is yet established,
     prefer molds that read naturally in the codebase's working language
     (e.g., "maximum number of points" reads more naturally as `max_points`
     than as `points_max`), and consider prepositional forms where they aid
     natural readability (e.g., `indexOf`, `elementAt`).
   - Note that these steps don't have to happen strictly in order in
     practice (a word may come to mind before its concept is fully
     analyzed), but recommend deliberately checking all three before
     finalizing a name, since skipping the concept-selection step is what
     most often produces a name that's individually reasonable but
     inconsistent with its peers.

5. **Maintain a project lexicon to resolve synonym ambiguity**
   - Recommend a living lexicon documenting the team's chosen word for each
     recurring domain or programming concept, along with rejected
     synonyms — e.g., "use `remove`, not `delete`, for this operation
     across the codebase."
   - Use the lexicon as the first reference when reviewing a proposed name
     that introduces a new synonym for an already-named concept — recommend
     the established term instead, or, if the new term is genuinely better,
     recommend updating the lexicon (and, over time, the existing names)
     rather than letting both terms coexist.

---

## Decision points and guidance

- **Does this name's mold match its established peers?** If a structurally
  similar concept already has a name elsewhere in the codebase, check that
  the new name follows the same pattern, not just that it's individually
  clear.
- **Is this the first time this kind of concept is being named in this
  codebase?** If so, treat the choice as setting the mold going forward —
  worth a deliberate decision, not a quick pick.
- **Does the proposed name introduce a new synonym for an existing
  concept?** Check the project lexicon (or start one) before accepting it.
- **Was the concept-selection step actually done?** A name that's missing
  information a nearby comment is compensating for is a sign step 1 of
  Feitelson's model was skipped.

---

## Quality criteria

A strong naming-consistency review should confirm that:

- **molds are identified explicitly:** review feedback names the specific
  structural pattern at issue, not just "this name looks off,"
- **the codebase converges on few molds:** a small, deliberate set of molds
  is used per structural pattern, not whatever emerged organically,
- **new names follow Feitelson's three steps:** concept selection, word
  choice, and mold construction are all deliberately considered,
- **a project lexicon exists and is used:** synonym ambiguity is resolved
  against a documented standard, not left to individual preference,
- **early naming choices get mold-setting attention:** the first instance
  of a new kind of concept is treated as establishing a pattern others will
  follow.

---

## Review checklist

Use these questions during the review:

- [ ] Does this name's structural mold match how similar concepts are
      already named in this codebase?
- [ ] If this is the first instance of this kind of concept, has a
      deliberate mold been chosen (not just whatever came to mind)?
- [ ] Does this name introduce a new synonym for a concept that already has
      an established term?
- [ ] Would a nearby comment become unnecessary if the concept it explains
      were folded into the name itself?
- [ ] Is there a project lexicon this name should be checked against (or
      that should be started)?

---

## Example prompts

- "These two variables represent the same kind of value but are named
  completely differently — help me figure out which naming pattern to
  standardize on."
- "Audit this module's variable names and tell me how many different name
  molds are in use for the same kinds of concepts."
- "Help me name this new field using our project's existing conventions,
  not just whatever sounds fine on its own."

---

## Related guidance

This skill complements:

- code-review-naming-fundamentals
- code-review-naming-word-choice
- code-review-quality-and-hygiene
- architecture-standards-adoption
- python-clean-architecture-domain-modeling
