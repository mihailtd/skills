---
name: architecture-refactoring-decision-criteria

description: Instructs the AI assistant on the concrete triggers that favor refactoring now (small well-tested scope, complexity actively hindering development with a citable incident count, an upcoming requirement shift, performance, new-technology adoption), the recognized reasons not to (low churn, pending deprecation, insufficient safety net, speculative future-extendability alone, no realistic time to finish, novelty-driven motivation) — explicitly excluding lack of code ownership as a valid reason to defer — and the five-category risk checklist (serious regressions, dormant bug exposure, scope creep, unnecessary complexity, incomplete/abandoned refactor) to weigh for every candidate being recommended.
---

# Refactoring Decision Criteria Instructions

When supporting the decision of whether a specific refactoring candidate is
worth doing now, use this tool to apply a concrete, evidence-based set of
triggers, disqualifying reasons, and risks — rather than a gut call on
whether the code "looks bad."

---

## Purpose

This tool helps the AI assistant by:

- naming the specific, recognized triggers that favor refactoring a
  candidate now, so a recommendation cites a concrete reason rather than a
  vague "it's messy,"
- naming the specific, recognized reasons *not* to refactor a candidate
  right now, so a credible assessment can defer things explicitly and say
  why, not just recommend everything it finds,
- explicitly excluding a candidate's code ownership from that list of
  valid reasons to defer — lack of primary ownership is never, on its own,
  a reason to withhold a refactoring recommendation,
- providing a five-category risk checklist (serious regressions, dormant
  bug exposure, scope creep, unnecessary complexity, incomplete/abandoned
  refactor) to weigh explicitly for every candidate being recommended, not
  as a formality but as real factors that can flip a recommendation.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- cites a specific recognized trigger (not a generic impression) whenever
  it recommends refactoring a candidate now,
- cites a specific recognized reason whenever it defers a candidate,
  explicitly listing what it is *not* recommending, not just what it is,
- never treats "someone else wrote/owns this code" as a valid reason to
  defer a refactor, regardless of how tempting a shortcut that is,
- weighs all five risk categories for every recommended candidate, and
  lets a genuinely severe risk finding change the recommendation rather
  than being noted and ignored,
- distinguishes cost/benefit reasoning grounded in concrete categories
  (developer productivity, bug-isolation ease) from a vague "cleaner code"
  gesture that doesn't actually justify the investment.

---

## Instructions for the AI

1. **Recognize the concrete triggers that favor refactoring now**
   - **Small, well-tested scope** — little should hold this back; recommend
     it plainly unless the "improvement" is genuinely uncertain or the
     surface area is larger than it first appears.
   - **Complexity is actively hindering development right now** — the area
     provokes real hesitation or dread from contributors, or has already
     caused at least one traceable bug or incident. Where possible, cite a
     concrete count — bugs/incidents traceable to this area over a recent
     window (e.g., the last six months) — rather than just a vibe that
     "this file is scary."
   - **An upcoming shift in product requirements** — when new
     functionality doesn't fit the current shape of the code, recommend
     refactoring *first*, then building the new feature on top of the
     refactored code, rather than bolting the requirement onto the
     existing structure. This is usually the easiest refactor to get
     buy-in for, since it rides alongside already-planned feature work
     instead of competing with it.
   - **Performance** — a legitimate refactoring motivation on its own
     (producing the same outputs faster or with less memory is still
     "unchanged external behavior"). Distinguish a narrow, clever,
     deeply system-knowledge-dependent performance patch — especially one
     guarded by a long warning comment about fragile behavior — from a
     broad, well-isolated performance improvement; the former is not the
     same kind of sustainable win and shouldn't be prioritized the same
     way. Call out fragile-clever-patch-shaped performance code as its own
     finding.
   - **Adopting a new technology or library** — introducing something new
     into an already-tangled area sets the new thing up to fail; recommend
     cleaning up the code the new technology will touch *before or as
     part of* the adoption, not as an unrelated follow-up. The new
     technology itself still needs a real, measured problem behind it,
     independent of the cleanup opportunity — see
     **architecture-trend-adoption-discipline**.

2. **Build the cost/benefit case using concrete categories, not vague gestures**
   - **Developer productivity** — a messy hotspot costs the *most* time
     not on the engineer who already knows it, but on everyone who
     doesn't: newer team members, engineers from other teams, and anyone
     returning after a long gap all have to reconstruct an understanding
     before they can even start solving their actual problem. If the file
     is also a commonly-copied reference point, weigh that compounding
     cost explicitly.
   - **Bug-isolation ease** — tangled, dense code makes bugs slower to
     localize, not just slower to fix; a history of slow incident
     resolution or repeated patch-on-patch fixes is evidence the structure
     itself is part of the cost.
   - Weigh these against the refactor's actual cost: effort, risk (see the
     risk checklist below), and disruption to any in-flight work touching
     the same area.

3. **Weigh all five risk categories explicitly for every recommended candidate**
   - **Serious regressions** — highest where a candidate is simultaneously
     high-churn, complex, and poorly tested; state this risk plainly
     rather than letting the benefit case stand alone.
   - **Dormant bug exposure** — refactoring can surface a pre-existing bug
     that was simply never exercised in its current shape (a previously
     "impossible" code path becomes reachable once the structure changes).
     Treat this as an expected possible outcome for any candidate touching
     long-unexamined code, not a sign the refactor itself is at fault if
     it happens.
   - **Scope creep** — flag candidates whose natural boundary is fuzzy or
     likely to invite "while we're here" expansion.
   - **Unnecessary complexity** — flag if the *proposed* refactor itself
     looks over-designed relative to the problem (see
     **architecture-simplicity** / **architecture-flexibility-complexity-tradeoffs**)
     — trading one kind of mess for premature abstraction is not a win.
   - **Incomplete/abandoned refactor** — a refactor left half-finished is
     worse than the original mess: it leaves the codebase in an ambiguous
     state where a reader can't tell which shape is the "real" one, and
     invites new code to be built on the part that's about to be
     replaced. Only recommend a scope that can realistically be
     *completed*, not merely started, given the time actually available —
     if the ideal scope doesn't fit, recommend a smaller, fully-completable
     slice over a larger one likely to stall.

4. **Recognize the concrete reasons not to refactor right now**
   - Low churn — little ongoing cost to relieve.
   - An area slated for deprecation/replacement soon — refactoring it is a
     sunk-cost trap.
   - A refactor whose risk currently outweighs its benefit, or a
     genuinely untested tangled file that needs a safety net built
     *first*, as its own separate step, before a refactor is recommended
     at all.
   - **Refactoring purely for hypothetical future extendability**, with no
     concrete near-term win identified — a speculative "this might be
     easier to extend someday" is not sufficient justification on its
     own; this is the same premature-generality argument
     **architecture-simplicity** / **architecture-trend-adoption-discipline**
     make elsewhere, applied to refactoring motivation specifically.
   - **No realistic time to see it through to completion** — see the
     incomplete-refactor risk above; recommend scoping down to something
     finishable, or deferring, rather than recommending a scope that can't
     realistically land.
   - A change motivated mainly by novelty ("this would be fun to
     rewrite") rather than a measured cost — don't recommend a refactor
     whose primary justification is that it would be an interesting or
     enjoyable exercise for whoever executes it.

5. **Never treat lack of code ownership as a valid reason to defer**
   - **Explicitly exclude** "someone else wrote/owns this code" from the
     list of valid deferral reasons above. A credible refactoring
     assessment names things it is *not* recommending, and the reasons it
     gives should always be about the code's actual state (churn, risk,
     timing, deprecation) — never about who happens to be its primary
     maintainer.
   - This reflects a deliberate team-culture stance some teams hold
     explicitly: refactoring and keeping code tidy is treated as a
     continuous, collective responsibility across the whole codebase, not
     gated by "whose corner of the code this is." Where that stance
     applies, the only thing that actually matters when working outside
     one's primary area is **coordination, not restriction** — visibility
     (a reviewable change, looping in the team/owner) and understanding
     prior context (checking history before assuming a strange-looking
     line is simply wrong) are good practice; withholding the refactor
     itself is not the fix.

---

## AI decision guidance

When generating refactoring-decision guidance, keep these principles in
mind:

- **Cite a specific recognized trigger or deferral reason** — never a bare
  "it's messy" or "it's fine for now."
- **Weigh all five risk categories for every recommended candidate** — a
  severe finding should be able to change the recommendation, not just
  decorate it.
- **Lack of code ownership is never a valid deferral reason** — the valid
  reasons are all about the code's actual state, not who wrote it.
- **A credible assessment names what it's declining to recommend**, with
  a specific reason, not just what it endorses.

---

## Success criteria

A strong response should ensure that it:

- **names a specific trigger** for every recommended candidate,
- **builds the cost/benefit case with concrete categories** (developer
  productivity, bug-isolation ease), not a vague "cleaner code" claim,
- **weighs all five risk categories explicitly**, letting a severe one
  change the recommendation,
- **names a specific reason for every deferred candidate**, and never
  cites lack of ownership as that reason,
- **treats incomplete-refactor risk as seriously as regression risk** —
  only recommends scope that can realistically be completed.

---

## Example prompts for the AI

- "Is this file worth refactoring now, or should we leave it alone?"
- "We're about to add a feature that doesn't fit this code's current
  shape — does that change the refactoring calculus?"
- "What's actually risky about refactoring this particular function?"
- "I don't really own this part of the codebase — should that stop me
  from refactoring it?"

---

## Related guidance

Use this tool alongside:

- architecture-refactoring-scope-classification — classify Local vs.
  At-Scale first; it determines how much process this decision actually
  needs.
- architecture-code-degradation-diagnosis — diagnosing why a candidate
  degraded shapes the cost/benefit case this skill builds.
- architecture-trend-adoption-discipline — the deeper check for whether a
  new-technology-adoption trigger, or a refactor's chosen destination, is
  solving a real, measured problem rather than chasing a trend.
- architecture-simplicity / architecture-flexibility-complexity-tradeoffs
  — the premature-generality argument behind the "no speculative
  extendability" deferral reason and the "unnecessary complexity" risk.
- architecture-refactoring-plan-structure — once a candidate clears this
  decision, how to actually structure its plan, including a mandatory
  cleanup step tied directly to the incomplete-refactor risk here.
