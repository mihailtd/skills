---
name: code-review-naming-word-choice
description: Guides AI reviewers to evaluate specific naming choices using research findings — full words beat abbreviations and single letters for comprehension and bug-finding, but longer/multi-syllable names cost more to recall; most single-letter variables carry no reliably shared meaning beyond a handful of exceptions; camelCase edges out snake_case on accuracy at a small speed cost; and locations with poor naming correlate with higher bug density.
---

# Code Review: Naming Word Choice

This skill helps AI reviewers apply specific, research-backed findings when
judging *what* a name is actually made of — full words versus abbreviations
versus single letters, which single letters (if any) carry real shared
meaning, camelCase versus snake_case, and the empirical link between poor
naming and bug-prone code. It complements code-review-naming-fundamentals,
which covers the cognitive mechanics and when to review names at all.

---

## When to use this skill

Use this skill when you need to:

- decide whether a proposed name should use full words, an abbreviation, or
  a single letter,
- evaluate whether a single-letter variable name (beyond well-established
  loop counters) is safe to assume a reader will understand,
- settle or apply a camelCase-versus-snake_case convention decision,
  especially when starting something new versus working in an established
  codebase,
- use naming quality as a signal for where a codebase might be more
  bug-prone and worth extra review attention,
- give an author concrete guidance on trading off name clarity against name
  length/recallability.

---

## Outcome

Produce word-choice feedback that:

- recommends full dictionary words over abbreviations or single letters by
  default, since they measurably improve both bug-finding speed and code
  comprehension,
- flags when a longer, multi-syllable name may be trading comprehension for
  recallability, and suggests a balance rather than defaulting to maximal
  verbosity,
- treats single-letter variable names with real skepticism outside a small
  set of letters with genuinely shared meaning (loop counters, common type
  abbreviations), since most single letters carry no consensus meaning at
  all,
- applies camelCase-versus-snake_case guidance appropriately — favoring
  consistency with an existing codebase's established style over switching,
  while noting camelCase's measured comprehension edge for new projects
  without an established convention,
- uses locations of poor naming as a practical heuristic for where bugs
  might be hiding, without overstating the (correlational, not necessarily
  causal) evidence behind it.

---

## Instructions for the AI

1. **Default to full words over abbreviations or single letters**
   - Recommend full, dictionary-word-based identifiers as the default
     choice: research measuring both bug-finding speed and code
     comprehension consistently finds full-word identifiers outperform both
     abbreviations and single letters, while abbreviations show no
     meaningful speed advantage over single letters despite being harder to
     type out.
   - When flagging an abbreviation or cryptic short name, don't just say
     "expand this" — note the specific comprehension or bug-finding cost
     the research associates with the choice.

2. **Weigh word count and syllables against recallability**
   - Recognize the real tradeoff: longer, full-word names are easier to
     understand when read, but are measurably harder to recall accurately
     later — driven specifically by syllable count, not raw character
     length.
   - When a name is accumulating many prefixes, suffixes, or qualifiers,
     flag it for review — added qualifiers should demonstrably earn their
     keep in clarity, since each one adds recall cost. Prefer trimming to a
     name that's clear but not maximally decorated.
   - Don't recommend indiscriminate name-shortening to fix this — the
     comprehension benefit of full words is the larger, better-evidenced
     effect; use this tradeoff mainly to catch names that have grown
     needlessly long, not to justify abbreviating.

3. **Treat single-letter variables with real skepticism**
   - Recognize that only a small number of single letters carry genuinely
     shared meaning across programmers — clear consensus exists mainly for
     `i`, `j`, `k`, `n` (integers, typically loop indices or dimensions),
     `s` (string), and `c` (character). Beyond this small set, there is
     little to no real consensus on what a given letter implies, even
     though programmers commonly assume there is.
   - Be specifically wary of letters like `d`, `e`, `f`, `r`, `t`, and
     `x`/`y`/`z` — these are inconsistently associated with different types
     (e.g., `x`/`y`/`z` are used about as often for integers as for
     floating-point numbers) — don't assume a reader will correctly infer
     type or purpose from these without other context.
   - When reviewing a single-letter name outside the well-established set,
     recommend either a full word or an explicit, documented project
     convention — don't accept "it's obviously an X" as sufficient
     justification unless it's one of the letters with genuine consensus.

4. **Apply camelCase vs. snake_case guidance contextually**
   - Recognize the measured tradeoff: camelCase names show meaningfully
     higher identification accuracy, at a small (roughly half-second) speed
     cost compared to snake_case.
   - Recognize that training effects are real and asymmetric: programmers
     who are more practiced in one style get measurably worse, not just
     neutral, at the other style — this is a switching cost, not just a
     preference difference.
   - When a codebase already has an established convention, strongly favor
     consistency with it over "improving" individual names to a different
     style — the consistency benefit (see code-review-naming-consistency)
     outweighs the modest accuracy difference between the two styles.
   - When establishing a *new* convention with no prior codebase history to
     defer to, camelCase's accuracy edge is a reasonable tie-breaker to cite
     — but frame it as a minor factor, not a strong mandate, and expect
     language/ecosystem conventions (e.g., Python's snake_case, Java's
     camelCase) to typically take precedence anyway.

5. **Use poor naming as a heuristic for bug-prone code**
   - Recognize the empirical finding that locations in a codebase with poor
     naming practices are more likely to also contain bugs (as measured by
     static analysis tooling) — a statistically significant association,
     even though the causal mechanism isn't established (poor names and
     bugs may share a common cause, such as high implementation complexity
     or a rushed/novice author, rather than one directly causing the
     other).
   - Use this as a practical prioritization signal during review: treat
     clusters of poor naming as a prompt for closer scrutiny of the
     surrounding logic, not just a documentation nitpick.
   - Frame the benefit accurately when recommending naming cleanup: improved
     names aren't guaranteed to prevent bugs, but they make bugs easier to
     spot and fix, and they're a legitimate, evidence-backed place to focus
     review attention.

---

## Decision points and guidance

- **Is this name an abbreviation or single letter that could be a full
  word instead?** Default to recommending the full word unless it's an
  established, low-risk exception (loop counters, well-known conventions).
- **Is this single letter one of the few with real consensus meaning
  (i/j/k/n, s, c)?** If not, don't let it pass on the assumption that "it's
  obvious."
- **Is this name accumulating length primarily from qualifiers/prefixes?**
  Check whether each addition earns its clarity cost, given the recall
  penalty of added syllables.
- **Does this codebase already have an established case convention?** If
  yes, consistency wins over the camelCase/snake_case research finding. If
  no convention exists yet, camelCase's accuracy edge is a reasonable
  tie-breaker.
- **Is there a cluster of poorly-named identifiers in this area?** Treat it
  as a signal to review the surrounding logic more carefully, not just the
  names.

---

## Quality criteria

A strong word-choice review should confirm that:

- **full words are the default:** abbreviations and single letters are used
  only where an established, low-risk exception applies,
- **length is purposeful:** longer names carry qualifiers that genuinely
  earn their added recall cost, rather than accumulating unnecessarily,
- **single letters are held to a real standard:** only letters with genuine
  shared meaning are used without further context or documentation,
- **case convention is applied consistently:** matches the existing
  codebase where one exists, or uses camelCase as a reasonable default
  where none exists yet and no language convention dictates otherwise,
- **naming quality informs review focus:** clusters of poor names are used
  as a prompt for closer scrutiny of nearby logic.

---

## Review checklist

Use these questions during the review:

- [ ] Are abbreviations or single letters used where a full word would be
      clearer, outside established exceptions?
- [ ] Do long names carry qualifiers that each add real clarity, or could
      some be trimmed without losing meaning?
- [ ] Are single-letter variables outside i/j/k/n/s/c used without a clear,
      documented meaning?
- [ ] Does the case convention (camelCase/snake_case) match the rest of the
      codebase?
- [ ] Are there clusters of poorly-named identifiers that warrant closer
      review of the surrounding logic?

---

## Example prompts

- "Is `d` a safe variable name here, or should I expect readers to be
  unsure what it means?"
- "This variable name has grown to five qualifiers — help me figure out
  which ones are actually earning their keep."
- "Should I use camelCase or snake_case for this new module, given the rest
  of the codebase is inconsistent?"

---

## Related guidance

This skill complements:

- code-review-naming-fundamentals
- code-review-naming-consistency
- code-review-quality-and-hygiene
- code-review-detect-bad-design
