---
name: code-review-linguistic-antipatterns
description: Guides AI reviewers to detect linguistic antipatterns — code where a name's implied behavior contradicts what the code actually does (a method that does more/less/the opposite of what it says, or an identifier that claims to contain more/less/the opposite of what it holds) — using Arnaoudova's six-category framework and concrete audit targets like "is"-prefixed non-booleans and setters that also return values.
---

# Code Review: Linguistic Antipatterns

This skill helps AI reviewers catch a specific, high-risk category of naming
problem: not names that are merely unclear, but names that actively
contradict what the code does. A method called `getCustomers` that returns a
boolean, or a variable called `isValid` that's actually an integer, doesn't
just fail to help the reader — it actively misleads them. This is a
distinct concern from name clarity or consistency (see
code-review-naming-fundamentals and code-review-naming-consistency): a name
can be perfectly well-formed and still lie about what it represents.

---

## When to use this skill

Use this skill when you need to:

- review method and function names against what they actually return or do,
- review identifier names (especially boolean-sounding and
  collection-sounding names) against what they actually hold,
- audit a codebase for specific, statistically common mismatch patterns
  (e.g., "is"-prefixed non-booleans, setters that also return values),
- explain why a misleading name is a more serious problem than a merely
  vague one,
- decide whether a name/comment or name/behavior contradiction found during
  review warrants a rename, a behavior change, or both.

---

## Outcome

Produce a linguistic-antipattern review that:

- checks method and identifier names against their actual behavior/content,
  not just their clarity in isolation,
- classifies any mismatch found into Arnaoudova's six antipattern
  categories, so feedback is specific about *how* the name misleads,
- explains the cognitive mechanism that makes these mismatches especially
  dangerous — the reader's long-term memory retrieves the wrong
  associations, and a plausible-sounding name lets the brain stop
  investigating before it discovers the truth,
- prioritizes auditing the specific patterns research has found to be
  common (e.g., "is"-prefixed non-booleans, dual-purpose setters), since
  these are worth systematic scanning rather than only opportunistic
  discovery,
- recommends fixing the mismatch by whichever side is wrong — sometimes the
  name should change, sometimes the behavior should be split so the
  original name becomes accurate again.

---

## Instructions for the AI

1. **Check names against actual behavior, not just clarity**
   - Recognize that a linguistic antipattern is fundamentally different
     from an unclear name: the name may be perfectly readable and
     well-formed, and still be *wrong* about what the code does. Review
     for this by comparing the name's implied contract against the actual
     implementation, not just judging the name in isolation.
   - Apply this check to method/function signatures, variable and field
     names, type names, and — per Arnaoudova's original framework — to
     comments as well, since a comment that contradicts the code it
     describes is the same category of problem in a different location.

2. **Classify mismatches into Arnaoudova's six categories**
   - **Methods that do more than they say** — e.g., a setter that also
     returns a value, or a `getX()` method that also mutates state.
   - **Methods that say more than they do** — e.g., a name implying a full
     collection operation (`getCustomers`) that actually returns a single
     item or a boolean.
   - **Methods that do the opposite of what they say** — e.g., a method
     named `validate()` that actually invalidates, or a comment describing
     behavior contrary to what the code performs.
   - **Identifiers that say they contain more than they do** — e.g., a
     name like `initial_element` that actually holds an index, not an
     element.
   - **Identifiers that say they contain less than they do** — e.g., a
     variable that appears to hold a single value but actually holds a
     collection or a compound structure.
   - **Identifiers that say the opposite of what they contain** — e.g., a
     boolean-sounding name (`isValid`) that actually holds a non-boolean
     value, such as an integer or a list of possible outcomes.
   - When flagging a finding, name the specific category — this makes the
     fix direction clearer (a "says more than it does" method needs either
     its name narrowed or its behavior expanded to match).

3. **Explain the specific cognitive mechanism at work**
   - **Wrong LTM retrieval:** a plausible but incorrect name activates the
     reader's prior knowledge about *that* kind of name — e.g., a name
     like `retrieveElements()` activates associations with collection
     operations (sort, filter, slice) that don't actually apply to a
     single-element return value. The reader isn't confused because the
     name is unclear; they're confused because the name successfully, but
     wrongly, informs them.
   - **Premature mischunking:** a name like `isValid` lets a reader
     conclude "this is a boolean" and stop investigating — there's no
     visible reason to dig further, so the misconception (that it's a
     boolean, when it's actually something else) can persist through
     multiple encounters with the code before it's ever corrected.
   - Use these mechanisms specifically to explain to an author *why* this
     category of issue is worse than a merely vague name: a vague name
     prompts a reader to look closer; a linguistically-antipatterned name
     actively discourages looking closer by supplying a confident, wrong
     answer.

4. **Prioritize auditing statistically common mismatch patterns**
   - When scanning a codebase (rather than reviewing a specific diff),
     prioritize checking these patterns first, since research has found
     them to occur at meaningfully high rates:
     - Setter methods that also return a value (found in roughly one in
       nine setters in studied codebases) — check whether the return value
       is meaningful or accidental, and whether callers rely on it in ways
       that aren't obvious from the method's apparent "just sets a field"
       contract.
     - Identifiers prefixed with `is` (or similar boolean-implying
       prefixes like `has`, `can`) that don't actually hold a boolean
       value — research has found roughly two-thirds of such names failed
       to hold an actual boolean in the codebases studied, making this one
       of the highest-yield patterns to check.
     - Method names and their accompanying comments that describe
       contradictory behavior — less common numerically, but especially
       dangerous since the comment is often trusted more than the code
       itself.
   - Recommend these as concrete, scriptable/greppable starting points for
     a systematic naming audit, not just something to notice opportunistically.

5. **Fix the mismatch on the correct side**
   - When a mismatch is found, determine whether the *name* or the
     *behavior* is the actual problem, and recommend fixing that side:
     - If the behavior is reasonable but the name overpromises or
       underpromises, rename to match what the code actually does (see
       code-review-naming-fundamentals and code-review-naming-consistency
       for how to construct the corrected name well).
     - If the name is the more sensible contract, but the implementation
       has grown extra, unexpected behavior (e.g., a setter that
       accumulated a return value nobody asked for), recommend separating
       the extra behavior into its own explicitly-named operation instead
       of leaving it silently bundled in.
   - Avoid recommending a fix that just makes the name vaguer to avoid the
     contradiction — that trades a misleading name for an uninformative
     one, losing the benefit of a name that's both accurate and clear.

---

## Decision points and guidance

- **Is this name unclear, or is it actively wrong?** A merely unclear name
  is a code-review-naming-fundamentals concern; a name that contradicts
  actual behavior is a linguistic antipattern — treat the latter with
  higher urgency, since it actively misleads rather than just failing to
  help.
- **Which of the six categories does this mismatch fall into?** Naming the
  category clarifies whether the fix should narrow the name, expand it, or
  correct a direct contradiction.
- **Is this one of the high-yield patterns (is-prefixed non-booleans,
  dual-purpose setters)?** If so, treat it as worth a systematic codebase
  scan, not just a one-off finding.
- **Should the name or the behavior change?** Judge which side actually
  reflects the intended contract, and fix the other side to match it.

---

## Quality criteria

A strong linguistic-antipatterns review should confirm that:

- **names are checked against real behavior**, not just judged for
  surface-level clarity,
- **every finding is categorized** using Arnaoudova's six-category
  framework, not left as a generic "confusing name" comment,
- **the cognitive mechanism is explained** (wrong LTM retrieval or
  premature mischunking) so the author understands why this is a
  higher-severity issue than an unclear name,
- **high-yield patterns get systematic attention:** `is`/`has`-prefixed
  identifiers and dual-purpose setters are checked as a matter of course,
  not just when they happen to be noticed,
- **fixes target the correct side** of the mismatch — renaming when the
  behavior is right, restructuring when the name is right, never just
  vaguing out the name to dodge the contradiction.

---

## Review checklist

Use these questions during the review:

- [ ] Does this method's name accurately describe everything it does (not
      more, not less, not the opposite)?
- [ ] Does this identifier's name accurately describe what it actually
      holds?
- [ ] For every `is`/`has`/`can`-prefixed identifier, does it actually hold
      a boolean?
- [ ] For every setter, does it return a value — and if so, is that
      intentional and documented?
- [ ] Does any comment describe behavior that contradicts what the code
      beneath it actually does?
- [ ] For any mismatch found, is the fix targeting the side (name or
      behavior) that's actually wrong?

---

## Example prompts

- "Audit this file for methods whose names promise something different
  from what they actually do."
- "Check every `is`-prefixed variable in this module and tell me which ones
  aren't actually booleans."
- "This setter also returns a value — is that intentional, and does the
  name still make sense given that?"

---

## Related guidance

This skill complements:

- code-review-naming-fundamentals
- code-review-naming-word-choice
- code-review-naming-consistency
- code-review-cognitive-load-smells
