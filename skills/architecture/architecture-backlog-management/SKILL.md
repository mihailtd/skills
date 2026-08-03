---
name: architecture-backlog-management
description: Instructs the AI assistant to run the architectural backlog as the team's external memory — capturing every idea immediately regardless of perceived importance, checking for duplicates before adding, retaining rejected items instead of deleting them, curating a small "up next" set, and running periodic housecleaning sweeps — plus maintaining software/data-model catalogs as the companion record of current system state.
---

# Backlog Management Instructions

When supporting architecture design, use this tool to help teams run their
architectural backlog as a genuine external memory system — capturing
every idea the moment it appears, keeping the backlog tidy and searchable,
never deleting rejected items, curating a small next-up set, and revisiting
the rest on a periodic cadence — alongside the companion practice of
maintaining software and data-model catalogs that document current system
state.

---

## Purpose

This tool helps the AI assistant by:

- framing the backlog's most valuable property as external memory: capture
  an idea, then let it go, trusting the backlog to remember it perfectly so
  attention can return to the task at hand,
- recommending immediate, low-friction capture of every idea regardless of
  its perceived importance — deferring evaluation rather than trying to
  judge relevance in the moment,
- insisting rejected items stay in the backlog (marked closed/resolved,
  never deleted) so that the team's collective history of what's been
  considered and why remains available,
- distinguishing the full backlog (potentially hundreds of items, mostly
  dormant) from the small "up next" set (five to ten items) that should
  always be ready for architects to pick up,
- recommending a periodic housecleaning cadence (a few times a year) with a
  specific set of review questions, so dormant items are neither forgotten
  nor left to silently rot,
- treating software and data-model catalogs as the backlog's companion
  artifact — documenting the system's *current* state rather than
  potential future changes — linked to related change proposals but not
  necessarily maintained in the same tool.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- captures every new idea into the backlog immediately, using it to defer
  and externalize evaluation rather than deciding relevance on the spot,
- checks for existing, related entries before adding a new one, and
  enriches an existing entry rather than creating a near-duplicate,
- writes backlog entries with enough context (a real paragraph, not a
  fragment) that they're still useful to someone reading them months later
  with no memory of the original conversation,
- never deletes a rejected item — marks it closed/resolved instead, so it
  remains available if the topic resurfaces,
- maintains a small, current "up next" list distinct from the full backlog,
  and reviews it on whatever cadence (planning cycle, roadmap, or a
  monthly curation pass) fits the team,
- runs a periodic (roughly quarterly) housecleaning sweep over dormant
  backlog items, asking explicitly whether each is still relevant, worth
  addressing now, and related to other items worth consolidating or
  linking,
- maintains software and data-model catalogs as living documentation of
  the system's current state, cross-linked to the change proposals that
  affect them.

---

## Instructions for the AI

1. **Treat the backlog as external memory, and use it that way**
   - When a new idea surfaces — whether during a change discussion, a
     tangential thought, or an interruption — recommend capturing it in the
     backlog immediately and then setting it aside, rather than trying to
     evaluate its importance in the moment.
   - Use "great idea — let's add it to the backlog" as the default response
     to redirect an in-the-moment idea or interruption; this both honors
     the idea and protects focus on the current task.
   - Don't pre-filter by perceived importance before capturing — critically
     unimportant-seeming and critically important-seeming ideas alike
     should go in; judgment is deferred to a later, less distracted moment
     (see the housecleaning sweep, step 4).

2. **Check for duplicates and write real descriptions**
   - Before adding a new item, recommend a quick search of the existing
     backlog for something equivalent — if found, enrich that entry
     (sharpen the argument, add detail) instead of creating a near-duplicate.
   - Push for a genuine paragraph of context per entry, not a fragment —
     the entry needs to be self-sufficient for someone (possibly the same
     person, months later) who has no memory of the situation that
     prompted it.

3. **Never delete a rejected item**
   - When a proposal or idea is rejected, recommend marking it
     closed/resolved rather than deleting it — the record of what was
     considered and why it didn't proceed is valuable collective memory,
     and revisiting the topic later should start from that context rather
     than from scratch.
   - If a closed topic resurfaces, use the existing entry's history to
     inform whether revisiting is actually worthwhile this time, and why.

4. **Curate a small "up next" set separately from the full backlog**
   - Recommend maintaining a short list — on the order of five to ten
     items — of what's coming up next, distinct from and much smaller than
     the full backlog, so architects always have a ready next task.
   - This can be derived from an existing planning cycle or roadmap if one
     exists; otherwise, recommend a periodic (e.g., monthly) curation pass
     specifically to keep this list current.

5. **Run a periodic housecleaning sweep over dormant items**
   - Recommend setting aside dedicated time a few times a year (roughly
     quarterly) to review every backlog item that isn't currently active or
     on the "up next" list — active and up-next items are already being
     looked at regularly and don't need this separate pass.
   - For each item reviewed, ask explicitly:
     - **Is this still relevant?** Close it if the system or context has
       moved on and made it obsolete.
     - **Do we want to address this now?** Promote it to "up next" if so,
       respecting the practical limit on how much can be active at once.
     - **What else relates to this?** Look for clusters of related items
       (sometimes from different sources) worth consolidating or linking.
   - Frame these reviews as reinforcing the backlog's core promise: filing
     something doesn't mean it's forgotten — it means it will be revisited.

6. **Also consult the backlog before planning exercises**
   - Recommend reviewing the backlog specifically before architectural or
     broader product/platform planning cycles — this is where previously
     captured ideas, if properly recorded, become genuinely useful inputs
     rather than needing to be reconstructed from memory.

7. **Maintain software and data-model catalogs as the companion artifact**
   - Recommend a software catalog recording the system's components
     (libraries, services, applications, frameworks, databases, etc.) and
     their relationships, with at minimum type, technology dependencies,
     and inter-component relationships captured per entry — links to
     documentation, owners, and runbooks add further value.
   - Recommend a data-model catalog recording the data/entity types and
     relationships the system operates on, using whatever modeling
     language, schema, or format specification fits the system.
   - Treat these catalogs as documentation of current state — a companion
     to the backlog's documentation of potential and past change, not a
     replacement for either.
   - These catalogs change less frequently and carry different metadata
     than the backlog, so they don't need the same tool — but do maintain
     links (e.g., via stable entry URLs) between catalog entries and the
     change proposals that affect them.

---

## AI decision guidance

When generating backlog-management guidance, keep these principles in mind:

- **Capture now, judge later:** always default to immediate, low-friction
  capture over in-the-moment evaluation.
- **Never delete, always close:** rejected or obsolete items lose their
  historical value if deleted — recommend the closed/resolved state
  instead.
- **Two lists, not one:** keep the small, current "up next" set visibly
  distinct from the full, mostly-dormant backlog.
- **Schedule the review, don't rely on memory:** dormant items need a
  periodic sweep with the three specific questions (relevant? address now?
  related to what?) — don't assume they'll surface on their own.
- **Catalogs document now, backlog documents next:** keep these two artifact
  types conceptually distinct even when cross-linking them.

---

## Success criteria

A strong backlog-management response should help teams achieve:

- **frictionless capture:** every new idea recorded immediately, regardless
  of its perceived importance,
- **a tidy, deduplicated backlog:** related entries linked or merged rather
  than scattered as near-duplicates,
- **a durable historical record:** rejected items retained and marked
  closed, available if the topic resurfaces,
- **a ready next-up set:** a small, current list of what's coming next,
  distinct from the full backlog,
- **a working housecleaning cadence:** dormant items periodically reviewed
  against relevance, priority, and relatedness,
- **current-state catalogs:** software and data-model catalogs maintained
  and cross-linked to relevant change proposals.

---

## Example prompts for the AI

- "Someone just mentioned an idea in passing during our design review —
  help me capture it properly in the backlog so we don't lose it."
- "Our backlog has grown to hundreds of items and feels unmanageable — help
  me set up a periodic review process for it."
- "We rejected a proposal last year and someone wants to revisit it — how
  should we use our backlog history to inform that conversation?"
- "Help me set up a software catalog that documents our system's current
  components and links back to the change proposals that touched them."

---

## Related guidance

Use this tool alongside:

- architecture-change-proposals
- architecture-change-templates
- architecture-review-process
- architecture-velocity-estimation
