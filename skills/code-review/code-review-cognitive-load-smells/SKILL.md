---
name: code-review-cognitive-load-smells
description: Guides AI reviewers to flag Fowler-catalog code smells (long parameter lists, primitive obsession, data clumps, temporary fields, shotgun surgery, and more) by explaining the specific cognitive-load mechanism each one triggers — working-memory overload, failed chunking, or mischunking — and using the Paas Scale as a quick self-assessment technique. Complements code-review-code-structure and code-review-detect-bad-design, which already cover feature density, cyclic dependencies, and mesh/bossy components.
---

# Code Review: Cognitive Load Smells

This skill helps AI reviewers apply Fowler's code smell catalog with an
explicit cognitive mechanism behind each flag — not just "this pattern is
bad," but *which* cognitive process it overloads (working memory capacity,
chunking via meaningful names, or reliable pattern-matching between similar
code). This skill focuses on the smells and cognitive mechanisms not already
covered by code-review-code-structure and code-review-detect-bad-design
(feature density, cyclic dependencies, mesh/bossy components) — use this
skill alongside those, not instead of them.

---

## When to use this skill

Use this skill when you need to:

- explain *why* a flagged code smell is a real comprehension problem in
  terms a skeptical author will find concrete, rather than citing "best
  practice" alone,
- review method signatures, class boundaries, and codebase-wide patterns
  against Fowler's smell catalog, specifically the smells centered on
  primitive types, data grouping, temporary state, and change locality
  that aren't already covered by this package's structural-smell skills,
- decide whether a method's parameter list or a conditional structure has
  crossed from acceptable into cognitively overloading,
- quickly estimate whether a piece of code imposes too much cognitive load,
  using a lightweight self-assessment technique,
- justify prioritizing a smell-driven refactor by citing its correlation
  with real bug and change risk, not just readability.

---

## Outcome

Produce a review that:

- names the specific cognitive mechanism behind each flagged smell —
  working-memory overload, failed chunking, or mischunking — so the author
  understands the actual reader cost, not just a label,
- applies the working-memory capacity limit (roughly six chunks) as a
  concrete, explainable threshold for parameter lists and branching
  complexity, while accounting for cases where related parameters form a
  single conceptual chunk rather than several,
- flags Fowler smells not already covered elsewhere in this package:
  primitive obsession, data clumps, temporary field, alternative classes
  with different interfaces, incomplete library class, parallel
  inheritance, refused bequest, divergent change, shotgun surgery,
  speculative generality, message chains, middle man, and inappropriate
  intimacy,
- treats code clones specifically as a mischunking risk (the brain
  pattern-matches similar-looking code as identical), not just a
  duplication/maintenance concern,
- offers the Paas Scale as a fast, practical technique for a reviewer or
  author to self-rate a piece of code's cognitive load,
- cites the empirical link between code smells and both error-proneness and
  change-proneness as justification for prioritizing smell cleanup, without
  overstating it as guaranteed bug prevention.

---

## Instructions for the AI

1. **Ground each smell in its specific cognitive mechanism**
   - **Working-memory overload** (long parameter lists, complex switch/
     conditional statements): the working memory holds roughly six chunks
     at once. A parameter list beyond that count is likely to exceed
     capacity — but check whether groups of parameters form a single
     conceptual chunk to a reader who already has the relevant domain
     knowledge (e.g., `xOrigin, yOrigin, xDestination, yDestination` reads
     as two chunks — "origin" and "destination" — not four). Flag the
     smell based on effective chunk count, not raw parameter count.
   - **Failed chunking** (long method / God method, large class / God
     class, lazy class): well-named functions and classes let a reader
     treat a whole block of logic as a single, already-understood chunk
     (calling `square(5)` doesn't require reading its implementation). A
     method or class that's grown too large no longer offers this
     shortcut — there's no name that can stand in for "does twelve
     unrelated things," forcing the reader back to line-by-line
     comprehension. Frame this as the loss of a chunking shortcut, not
     just "this is long."
   - **Mischunking** (duplicated code / code clones): when a reader
     encounters code that looks similar to something they already
     understand, their brain will often pattern-match it as the *same*
     thing rather than inspecting the differences closely — the more
     similar the name and shape, the stronger this false-equivalence
     effect. Flag clones specifically as a misconception risk (a reader
     may genuinely believe two clones behave identically when they don't),
     not only as a maintenance-burden concern, and note that this kind of
     misconception can persist through multiple exposures before it's
     corrected.

2. **Apply the parameter-list and complexity threshold concretely**
   - Recommend treating roughly six chunks as the practical ceiling for a
     parameter list or a branching structure a reader needs to hold in mind
     at once.
   - Before flagging a long parameter list, check whether several
     parameters can be viewed as one conceptual group by a reader with
     relevant domain knowledge — if so, the effective chunk count may be
     well under the raw parameter count, and the smell may not actually
     apply. When it doesn't group naturally, recommend consolidating
     related parameters into a single parameter object as the fix, which
     both reduces the raw count and gives the group a meaningful name.

3. **Check for the Fowler smells not already covered by this package's other skills**
   - **Primitive obsession** — overuse of primitive types (raw strings,
     numbers) where a small dedicated type would better represent a
     domain concept and prevent misuse.
   - **Data clumps** — the same group of values (e.g., a street, city, and
     postal code) repeatedly passed around together without being unified
     into a class or structure.
   - **Temporary field** — a field that's only meaningfully set/used in
     some code paths, leaving readers to track when it's actually valid.
   - **Alternative classes with different interfaces** — two classes doing
     essentially the same job with different method/field names, forcing
     a reader to learn two vocabularies for one concept.
   - **Incomplete library class** — methods bolted onto unrelated classes
     because a library or shared class couldn't be extended directly.
   - **Parallel inheritance** — every subclass of one class requiring a
     matching subclass of another, suggesting the two hierarchies should
     really be one.
   - **Refused bequest** — a subclass that inherits behavior it doesn't
     actually use, signaling the inheritance relationship itself may be
     wrong.
   - **Divergent change** and **shotgun surgery** — respectively, one
     class needing to change for many unrelated reasons, and one logical
     change requiring edits across many scattered classes; both indicate
     the code's structure doesn't match how it actually changes over time
     (see architecture-composition for the general principle that
     relationships should minimize this kind of scattered impact).
   - **Speculative generality** — code, hooks, or abstraction added "in
     case it's needed later" with no current use (see
     architecture-simplicity's caution against future-proofing for the
     same underlying concern at the architecture level).
   - **Message chains** — long chains of calls reaching through several
     objects to get to what's actually needed, forcing the reader to
     trace a long path just to find the relevant operation.
   - **Middle man** — a class that delegates so much of its behavior
     elsewhere that it's no longer adding meaningful value itself.
   - **Inappropriate intimacy** — two classes reaching into each other's
     internals so extensively that they're effectively coupled as one,
     without admitting it structurally.
   - **Flag argument** — a boolean (or small-enum) parameter that
     branches a function's behavior into genuinely different cases (e.g.
     `is_png`, `use_average`), especially where a second or third flag has
     been added since the first (`is_png` → `is_png, is_gif`). This is a
     mischunking risk at the call site specifically: `upload(file, true,
     false)` gives a reader no way to know what the flags mean without
     cross-referencing the function definition, and it signals a function
     that's grown to do several distinct things by picking the cheapest
     extension point each time rather than splitting. Recommend splitting
     into purpose-specific functions (a shared pre-/post-processing helper
     plus one function per case) instead of adding another flag.
   - For each, flag it with the specific reader cost it causes (extra
     indirection to trace, an extra vocabulary to learn, an extra
     invisible precondition to track) rather than citing the smell name
     alone as sufficient justification.

4. **Use the Paas Scale as a lightweight cognitive-load self-check**
   - Recommend a simple self-rating technique: after reading a piece of
     code cold, rate the mental effort required on a 1-9 scale (from "very,
     very low" to "very, very high" mental effort).
   - Use a consistently high self-rating across reviewers as a practical
     trigger for investigating *why* — cross-reference against the smells
     in this skill to identify the specific structural cause, rather than
     leaving the finding as a vague "this feels hard to read."
   - Treat this as a fast, low-overhead technique appropriate for spot
     checks during review, not a replacement for the more rigorous
     structural analysis in step 1-3.

5. **Cite the real, evidence-backed stakes without overstating them**
   - When justifying smell cleanup, note that empirical studies of
     production codebases have found God classes and God methods to be
     significant contributors to error-proneness, and large classes and
     long methods to be significantly more likely to require future
     changes than non-smelly code.
   - Be precise about what this evidence supports: code smells correlate
     with higher bug and change risk, which is a strong prioritization
     signal — but a smell's presence doesn't itself guarantee a bug is
     present, and removing a smell doesn't guarantee bugs are prevented.
     Frame recommendations accordingly (a reason to prioritize scrutiny,
     not a promise of a specific outcome).

---

## Decision points and guidance

- **Does this smell overload working memory, break chunking, or risk
  mischunking?** Identify the specific mechanism before writing the
  finding — it changes both the explanation and the fix.
- **Do these parameters form a natural conceptual group?** If yes, the raw
  count may overstate the actual cognitive load; if no, recommend
  consolidating into a parameter object.
- **Is this smell already covered by code-review-code-structure or
  code-review-detect-bad-design?** If it's feature density, cyclic
  dependencies, or mesh/bossy components, defer to those skills instead of
  duplicating the finding here.
- **Is this a code clone, or a legitimate near-duplicate?** Check whether
  the similarity is likely to cause a reader to falsely assume identical
  behavior — that's the specific risk worth flagging, not similarity alone.
- **Would consolidated evidence (Paas Scale ratings, error/change-proneness
  data) strengthen this finding's priority?** Use it to justify urgency
  when available, without treating correlation as guaranteed causation.

---

## Quality criteria

A strong cognitive-load-smells review should confirm that:

- **every finding names its mechanism:** working-memory overload, failed
  chunking, or mischunking — not a bare smell label,
- **parameter/complexity thresholds are applied with judgment:** raw counts
  are checked against natural conceptual grouping before being flagged,
- **the less-common Fowler smells get real coverage:** primitive obsession,
  data clumps, temporary fields, parallel inheritance, refused bequest,
  message chains, middle man, and inappropriate intimacy are checked, not
  just the commonly-cited long-method/large-class smells,
- **code clones are flagged as a misconception risk**, not only a
  duplication/maintenance concern,
- **evidence is cited accurately:** correlational error/change-proneness
  findings are used to prioritize, not oversold as causal guarantees.

---

## Review checklist

Use these questions during the review:

- [ ] Does any method have a parameter list exceeding roughly six effective
      chunks (after accounting for natural grouping)?
- [ ] Does any method or class lack a name that could stand in for its full
      implementation (i.e., has it outgrown its chunking shortcut)?
- [ ] Are there near-duplicate code blocks that a reader might mistake for
      identical behavior?
- [ ] Is there primitive obsession, a data clump, or a temporary field that
      should be consolidated into a proper type or structure?
- [ ] Are there two classes doing the same job with different interfaces?
- [ ] Does a change to one concept require edits scattered across many
      files (shotgun surgery), or does one class change for many unrelated
      reasons (divergent change)?
- [ ] Is there speculative generality — abstraction with no current use?
- [ ] Are there long message chains, a middle-man class, or inappropriately
      intimate classes reaching into each other's internals?
- [ ] Does any function take a boolean/flag parameter that branches its
      behavior into genuinely different cases, especially one that's
      multiplied over time (a second or third flag added since the first)?

---

## Example prompts

- "This method has nine parameters — is that actually a problem, or do they
  form natural groups a reader would chunk together?"
- "Explain why this large class is hard to read in terms of what it costs a
  reader's memory, not just 'it's too big.'"
- "These two functions look almost identical — could someone reading this
  code mistake one for the other?"

---

## Related guidance

This skill complements:

- code-review-code-structure
- code-review-detect-bad-design
- code-review-linguistic-antipatterns
- architecture-simplicity
- architecture-composition
