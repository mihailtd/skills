---
name: code-review-naming-fundamentals
description: Guides AI reviewers to evaluate identifier names using cognitive-science research — why names matter disproportionately, how formatting supports short-term memory while word choice activates long-term memory, why names should be evaluated at review time rather than while coding, and why early naming decisions in a project tend to stick for its lifetime.
---

# Code Review: Naming Fundamentals

This skill helps AI reviewers treat identifier naming (variables, functions,
classes, types, modules) as a first-class code quality concern grounded in
research on how programmers cognitively process code, not just a style
preference. It explains why names matter more than their small footprint
suggests, how good and bad names interact with a reader's memory, and when
in the development process naming quality should actually be evaluated.

---

## When to use this skill

Use this skill when you need to:

- review identifier names (variables, functions, methods, classes, types,
  modules) as part of a code review,
- explain to an author *why* a flagged name is a real comprehension problem,
  not just a style nitpick,
- decide when naming quality should be assessed — during active coding, or
  at a separate point in the process,
- set naming expectations early in a new project or codebase, since initial
  practices tend to persist for the codebase's lifetime,
- triage where in a codebase to look first for comprehension or bug risk
  driven by poor identifier choices.

---

## Outcome

Produce a naming review that:

- explains why a flagged name matters in terms of concrete reader cost
  (short-term memory chunking difficulty, or failure to activate relevant
  long-term memory), not just "this looks unclear,"
- distinguishes two legitimate but different naming philosophies —
  syntactic correctness and codebase-wide consistency — and applies both
  without treating either as sufficient on its own,
- recommends evaluating naming quality at review time rather than during
  active implementation, since naming decisions made under high cognitive
  load are unreliable,
- treats a new project's or new module's first names as high-leverage,
  since research shows naming quality set early in a codebase's life tends
  to persist rather than improve over time,
- gives specific, example-grounded feedback rather than vague "improve this
  name" comments.

---

## Instructions for the AI

1. **Treat names as a disproportionately large part of the codebase**
   - Remember that in real codebases, identifier names make up a large share
     of what's actually read — not just a small stylistic detail, but a
     substantial fraction of a file's tokens and characters.
   - Weight naming feedback accordingly during review: it is not a minor
     concern relative to logic or structure, it's a major lever on how
     readable the surrounding code actually is.
   - Recognize the other reasons naming quality compounds: it's one of the
     most common topics raised in code reviews, it's the most accessible
     form of documentation (available right in the code, unlike external
     docs), and clear names act as beacons that help an unfamiliar reader
     get oriented quickly.

2. **Ground feedback in how names interact with memory**
   - **Short-term memory (STM) / chunking:** a name's *formatting* — how
     clearly it splits into distinguishable parts — determines how easily a
     reader's STM can process it. A name like `nmcntravg` forces effortful,
     ambiguous parsing; a name like `name_counter_average` is chunked almost
     for free, despite being roughly twice as long. When flagging a
     cramped, unsegmented name, explain the problem in these terms.
   - **Long-term memory (LTM) / activation:** a name's *word choice*
     determines whether it successfully activates a reader's relevant prior
     knowledge. Names can draw on three distinct kinds of LTM knowledge:
     domain concepts (e.g., "customer," "shipment" — pulling in
     domain-specific associations), programming concepts (e.g., "tree,"
     "queue" — pulling in known behaviors and operations), and established
     conventions (e.g., `i`/`j` as loop counters, `n`/`m` as dimensions).
     When a name fails to draw on any of these, or uses a word that
     conflicts with the reader's existing associations, name that
     specifically as the failure mode rather than just calling the name
     "unclear."

3. **Apply both major naming perspectives, not just one**
   - **Syntactic correctness** (structural rules): watch for capitalization
     anomalies, consecutive or leading/trailing underscores, names that
     aren't real dictionary words (except well-established abbreviations),
     an unreasonable word count (recommend roughly two to four words;
     flag both single-word cryptic names and unwieldy five-plus-word
     names), overly short names outside a small conventional-exception list
     (e.g., loop counters like `i`, `j`, `k`), and type-encoding prefixes
     (Hungarian notation, e.g. `intPageCounter`).
   - **Codebase-wide consistency**: recognize this as an independent,
     equally valid lens — a name that is syntactically fine but breaks from
     how similar concepts are named elsewhere in the codebase is still a
     real problem, because it defeats chunking and LTM pattern-matching
     across the codebase. When the two perspectives conflict (a
     "syntactically better" name that would break local consistency),
     default to consistency with the existing codebase over introducing a
     locally-nicer but inconsistent name (see
     code-review-naming-consistency for the deeper treatment of this).

4. **Recommend evaluating naming quality outside active coding**
   - Recognize that naming decisions made while actively solving a problem
     are made under high cognitive load — the working memory is largely
     consumed by the problem itself, which is exactly why placeholder names
     like `foo` or `temp` get chosen in the moment. Don't treat this as
     carelessness; it's a predictable, structural effect of cognitive load.
   - Recommend that naming quality be assessed as a distinct activity,
     separate from implementation — code review is the natural point for
     this, since the reviewer isn't simultaneously solving the underlying
     problem.
   - When reviewing, use a naming-specific pass: list the identifier names
     introduced or changed, and for each, ask — is the meaning clear without
     other context? Is it ambiguous? Does it use a confusing abbreviation?
     Are similarly-named things actually similar in the code, and vice
     versa?

5. **Treat early naming decisions as high-leverage**
   - When starting a new project, module, or subsystem, apply extra
     scrutiny to the first names introduced — research shows naming
     practices tend to remain roughly constant within a codebase over time
     rather than improving as the codebase matures, so whatever pattern
     gets set early is likely to persist for the codebase's lifetime.
   - Use this as justification for investing more reviewer attention in
     naming during a project's early stages, rather than assuming it can be
     cleaned up incrementally later.

---

## Decision points and guidance

- **Is the flagged name hard to parse, or hard to relate to?** Distinguish
  formatting/chunking problems (STM) from word-choice/association problems
  (LTM) — they call for different fixes.
- **Does this violate a syntactic rule, a consistency norm, or both?** Name
  which one explicitly; a fix for one doesn't necessarily fix the other.
- **Is this a new project/module, or an established one?** Apply more
  scrutiny to first-time naming decisions in new code, since they'll likely
  set the pattern going forward.
- **Was this name coined mid-implementation?** If so, expect it may be a
  placeholder chosen under cognitive load — flag it for review-time
  attention rather than assuming it reflects the author's real intent.

---

## Quality criteria

A strong naming-fundamentals review should confirm that:

- **flagged names come with a concrete reason:** STM/chunking or
  LTM/activation failure, not a vague "unclear" label,
- **both perspectives are applied:** syntactic correctness and codebase
  consistency are both checked, with consistency taking priority when they
  conflict,
- **review, not coding, is where naming gets judged:** naming feedback is
  gathered as a distinct review pass, not expected to be perfect at
  authoring time,
- **early code gets extra scrutiny:** new projects or modules receive
  focused naming attention since their conventions are likely to persist,
- **feedback is specific:** comments name the exact word, segment, or
  convention at issue, not just "please rename."

---

## Review checklist

Use these questions during the review:

- [ ] Without outside context, is it clear what each new/changed name means?
- [ ] Does the name's formatting clearly separate its conceptual parts (STM)?
- [ ] Does the name's word choice draw on relevant domain, programming, or
      convention knowledge (LTM)?
- [ ] Are there capitalization anomalies, stray underscores, or
      non-dictionary abbreviations?
- [ ] Is the word count reasonable (roughly two to four words) for what the
      name represents?
- [ ] Are type-encoding prefixes (Hungarian notation) present and
      unnecessary?
- [ ] Are similarly-purposed names in this codebase actually named
      similarly?
- [ ] If this is a new project/module, has extra care been given to the
      first names introduced?

---

## Example prompts

- "Review the identifier names in this diff and tell me which ones are
  likely to cause real comprehension problems, and why."
- "Is this variable name unclear because of its formatting, or because the
  words chosen don't connect to anything a reader would already know?"
- "We're starting a new service — help me set naming expectations now,
  since I've heard early conventions tend to stick."

---

## Related guidance

This skill complements:

- code-review-naming-word-choice
- code-review-naming-consistency
- code-review-quality-and-hygiene
- code-review-code-structure
- code-review-linguistic-antipatterns
