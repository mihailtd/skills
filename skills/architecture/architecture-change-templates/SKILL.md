---
name: architecture-change-templates
description: Instructs the AI assistant to structure change-proposal documents with a mandatory, standardized template — used as a checklist of concerns to consider (not a form requiring every field), scaled so small changes stay under an hour to document, and itself maintained as a standard subject to the same change process it supports.
---

# Change Templates Instructions

When supporting architecture design, use this tool to help teams adopt a
standard template for change-proposal documents — using it as a checklist
that prompts consideration of each relevant concern rather than a form that
demands every field be filled, scaling it so small changes stay quick to
document, and treating the template itself as a standard maintained through
the same change process it supports.

---

## Purpose

This tool helps the AI assistant by:

- explaining why a common template has a multiplicative productivity
  effect: it accelerates writing (no time spent inventing structure),
  accelerates review (reviewers spend effort on the design, not on
  deciphering document organization), and — most valuably — acts as a
  built-in checklist that prompts the author to address each expected
  concern,
- providing the concrete section structure a change-proposal template
  should contain, and which sections apply at which stage of a change
  (motivation/concept versus detailed design),
- reinforcing that a checklist is not a form — sections that don't apply to
  a given change should be acknowledged as not applicable, not forced,
- insisting the template scale to the size of the change — a small,
  well-understood change should be documentable in under an hour, while
  larger changes are supported by repeating the detailed-design section
  per major element,
- recommending the template be adopted as a mandate, not an option, and
  treated as a standard maintained via the team's own change process (see
  architecture-change-proposals) — if it's not working, the fix is to
  revise the template, not to skip it.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- documents every change proposal (and, where useful, other artifacts like
  vision papers or catalog entries) against a shared, standard template
  rather than an ad hoc structure invented per document,
- uses the template as a prompt to actively consider each listed concern,
  explicitly marking non-applicable sections rather than either omitting
  them silently or padding them to look complete,
- keeps small, well-understood changes fast to document — under about an
  hour — while still supporting large, multi-element changes through
  repeated detailed-design sections,
- treats template adoption as mandatory across the team, since inconsistent
  use forfeits most of its benefit (both as an accelerant and as a
  checklist),
- revises the template itself through the normal change process when it
  isn't working well for someone, rather than tolerating ad hoc
  workarounds or silent non-compliance.

---

## Instructions for the AI

1. **Explain the multiplicative value of a shared template**
   - Make the case concretely: templates save the author from inventing
     document structure, freeing attention for the actual design content;
     they reduce reviewer cognitive overhead by making every document's
     shape familiar; and, most importantly, a well-designed template acts
     as a checklist that prompts the author to actually address each
     concern the team cares about (security, privacy, dependability, and
     so on).
   - Frame this as a multiplicative effect specifically because large
     projects generate many such documents — a small efficiency gain per
     document compounds significantly over hundreds or thousands of them.

2. **Structure the template around the change proposal's stage**
   - **Always required, from the motivational/conceptual stage onward:**
     - **Status** — placed at the top, so readers can immediately tell
       whether a document is current, outdated, in progress, or abandoned.
     - **Summary** — a concise statement of the problem being solved and
       the approach to solving it; complete enough that the rest of the
       document is genuinely "just the details."
   - **Added once the change reaches the detailed stage:**
     - **Terminology** — define any new terms the design introduces (don't
       re-document terms already captured in existing project
       terminology).
     - **Detailed Design** — the substantive section, repeated once per
       major design element; if a document needs much more than about
       half a dozen instances of this section, treat that as a signal the
       change should probably be split (see architecture-incremental-delivery).
     - **Dependability** — the design's reliability, resiliency,
       performance, and scale posture, and how it's achieved.
     - **Security and Privacy** — applicable concerns and how they're
       addressed.
     - **Efficiency** — for cloud-relevant designs, how unit cost behaves
       as usage scales (stays flat, grows, or shrinks).
     - **Compatibility** — how the change interacts with existing software
       and data, both within the system (migration feasibility) and for
       external clients depending on existing interfaces.
     - **Impacts** — a list or table of components/artifacts affected;
       this section summarizes information stated elsewhere and should
       introduce nothing new.
     - **Signatures** — recorded names of approvers, to encourage them to
       take the responsibility seriously (see architecture-review-process).

3. **Use the template as a checklist, not a form**
   - Make explicit that not every section needs content for every
     document — e.g., a new component with no prior version has no
     meaningful compatibility concerns to document.
   - The purpose of each section is to force a moment of consideration; if
     a concern genuinely doesn't apply, note that and move on rather than
     padding the section or treating its absence as an incomplete document.

4. **Scale the template to the change's size**
   - Recommend calibrating the template so that documenting a small,
     well-understood change takes under an hour — if it doesn't, the
     template is imposing unnecessary burden on small changes and should be
     revised.
   - For larger changes, rely on the repeatable Detailed Design section
     structure to scale up, rather than inventing a separate, heavier
     template for big changes.

5. **Extend the template pattern beyond change proposals where useful**
   - Recommend applying the same templated-checklist approach to other
     recurring artifact types the team produces — vision papers, catalog
     entries, and similar — wherever a consistent structure would offer the
     same writing/review/completeness benefits.

6. **Mandate the template and govern it through the normal change process**
   - Recommend making template use mandatory, not optional — a checklist
     used inconsistently loses most of its value, both as an accelerant and
     as a completeness check.
   - Treat the template itself as a standard the team maintains (see
     architecture-standards-adoption): if it's too burdensome or missing
     something, the fix is to propose a revision through the normal change
     process (using the template to document that very change), not to
     let individuals silently deviate from it.

---

## AI decision guidance

When generating change-template guidance, keep these principles in mind:

- **The template's real value is as a checklist, not as formatting:**
  emphasize the "did we consider X" function over mere document
  consistency.
- **Non-applicable is a valid answer, blank is not:** a section can be
  marked not applicable; it shouldn't be silently missing or artificially
  padded.
- **Scale down for small changes, don't create a second template:** use the
  same template's repeatable/optional structure to handle both small and
  large changes rather than maintaining parallel "lightweight" and
  "full" templates.
- **Mandatory, or it doesn't work:** inconsistent adoption undermines both
  the acceleration and the completeness-checking value — always recommend
  making it the standard, not a suggestion.
- **The template evolves like everything else:** route template complaints
  and improvements through the normal change process rather than tolerating
  ad hoc non-compliance.

---

## Success criteria

A strong change-template response should help teams achieve:

- **faster authoring and review:** documents that don't require reinventing
  structure each time, and reviewers who can focus on content over form,
- **genuine completeness checking:** each relevant concern (dependability,
  security/privacy, efficiency, compatibility, impacts) actively
  considered, with non-applicable sections explicitly marked rather than
  silently skipped,
- **stage-appropriate scope:** motivational/conceptual proposals kept to
  Status and Summary; detailed proposals expanded with the remaining
  sections as needed,
- **proportional effort:** small changes documentable in under an hour;
  larger changes scaled via repeated Detailed Design sections rather than a
  different process,
- **consistent, mandatory adoption:** the template used uniformly across
  the team, with revisions handled through the standard change process
  rather than informal workarounds.

---

## Example prompts for the AI

- "Help me set up a change-proposal template for our team that covers the
  concerns we consistently forget to address."
- "This section of our template doesn't apply to the change I'm
  documenting — is it okay to just note that and move on?"
- "Our template is taking too long to fill out for small changes — how
  should we scale it down without losing its checklist value?"

---

## Related guidance

Use this tool alongside:

- architecture-change-proposals
- architecture-backlog-management
- architecture-review-process
- architecture-standards-adoption
