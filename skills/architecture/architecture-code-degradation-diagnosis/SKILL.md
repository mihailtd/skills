---
name: architecture-code-degradation-diagnosis
description: >-
  Instructs the AI assistant to diagnose why code degraded — a requirement shift (the world around otherwise-reasonable code changed: scale, a product requirement, an external dependency, the environment/security landscape, or an unremoved deprecation) versus tech debt (a deliberate shortcut under time/resource pressure whose cost has since compounded) — before designing what a refactor should turn it into, since the diagnosis changes the correct target shape, not just the justification, and using git history plus consulting the original authors (a consult for context, never a gate on who may act) as the concrete diagnostic technique.
---

# Code Degradation Diagnosis Instructions

When supporting an assessment of code that looks bad today — tangled,
outdated, hard to extend — use this tool to diagnose *why* it degraded
before recommending what it should become. The diagnosis changes the
correct target shape of a refactor, not just how compelling the
justification sounds.

---

## Purpose

This tool helps the AI assistant by:

- establishing that code degrades when its perceived utility decreases —
  it stops behaving as well as it used to, or stops being as easy to read
  or extend — and that code which looks bad today was very often good
  code when it was written,
- distinguishing two root causes with genuinely different correct fixes: a
  **requirement shift** (the world around the code changed) and **tech
  debt** (a deliberate shortcut under real constraints whose cost has
  since compounded),
- providing a concrete diagnostic technique — checking version-control
  history and, where practical, consulting the people who wrote or last
  touched the code — rather than guessing or defaulting to whichever
  diagnosis is more convenient,
- reframing tech debt away from a blame-shaped narrative ("whoever wrote
  this was careless") toward an accurate one (a reasonable trade-off under
  real constraints, whose cost has since compounded), since the former
  produces worse team dynamics and doesn't actually improve the fix.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- diagnoses degraded code as either a requirement shift or tech debt
  before designing the refactor's target state, rather than jumping
  straight from "this looks bad" to a redesign,
- for a requirement-shift diagnosis, designs the target state around the
  *current* requirement — capturing what's been learned since the code was
  written — rather than producing a merely tidier version of the original
  design that still doesn't fit what's actually needed now,
- for a tech-debt diagnosis, actually pays down the shortcut properly
  rather than just reorganizing it, while still naming the shortcut's
  original context fairly (a reasonable call under real constraints, not
  carelessness),
- uses concrete diagnostic evidence — commit history, direct context from
  the original authors — instead of assuming a diagnosis, and records
  when the diagnosis is genuinely unclear rather than forcing one,
- treats consulting original authors as a way to get the diagnosis right,
  never as seeking permission to act — collective ownership of the
  codebase and getting the historical context right are compatible, not in
  tension.

---

## Instructions for the AI

1. **Recognize that degraded code was very often good code once**
   - Code degrades when its perceived utility decreases: it stops
     behaving as well as it used to, or stops being as easy to read or
     extend. This is a statement about the code's *current* fit, not a
     verdict on the original authors' skill or judgment.
   - Resist the urge to refactor immediately on encountering bad-looking
     code — doing so risks over-emphasizing whatever's most viscerally
     frustrating on first read, rather than addressing the code's true,
     underlying pain point. Diagnose first.

2. **Distinguish requirement shift from tech debt — they call for different fixes**
   - **Requirement shift**: the world around the code changed. This
     includes scale growing past what the code was designed for, a
     product requirement evolving, an external dependency deprecating the
     API surface the code relies on, the environment or security
     landscape moving (an OS/runtime change, a newly disclosed
     vulnerability class), or a feature being deprecated but its code
     never actually removed. The original code was a reasonable response
     to the requirements *at the time* — the right fix captures what's
     been learned since, not just "make it cleaner."
   - **Tech debt**: corners were deliberately cut under time or resource
     pressure, and the shortcut's cost has since compounded. The original
     trade-off may well have been the right call at the time — shipping
     fast can matter more than a clean abstraction — but the right fix is
     to actually pay down the shortcut properly, not just reorganize it
     around the same underlying gap.
   - Tech debt's common root causes are limited time, limited engineers,
     and limited money — every codebase has some, regardless of company
     age or size. Frame a tech-debt finding as a cost that's since
     compounded, not as evidence the original author was careless; this is
     both more accurate and more useful for deciding what to do next.
   - Don't default to assuming degraded code is tech debt. Misdiagnosing a
     requirement-shift case as tech debt produces a refactor that's
     cleaner but still doesn't fit what the code actually needs to do now
     — the underlying mismatch survives the refactor.

3. **Diagnose it with evidence, don't guess**
   - Check version-control history (commit messages, the timeline of
     changes to the area) for context around when the code was introduced
     or last significantly changed.
   - Where practical, ask the people who wrote or last touched the code.
     This is a *consult for context*, not a gate on who's allowed to act —
     collective refactoring and getting the diagnosis right are not in
     tension; asking is how the fix ends up correctly targeted, not a
     request for permission.
   - Listen for the signal in how people describe it: "we didn't know
     that would happen" or "at the time we assumed X" signals a
     requirement shift; "that was always a rushed job" or "we were just
     trying to hit a deadline" signals tech debt. Record which one
     applies — or that it's genuinely unclear — as part of the evidence
     for the refactor, not as an aside.

4. **Build the cost/benefit case to match the actual diagnosis**
   - For a **tech-debt** candidate, the case is comparatively
     straightforward: name what was skipped at the time and what it's
     costing now that the shortcut has compounded.
   - For a **requirement-shift** candidate, name specifically what
     changed — scale, a product requirement, an external dependency, the
     environment/security landscape, or an unremoved deprecation — and
     design the target state around the *current* requirement. A
     refactor that only tidies the existing shape without addressing what
     actually changed has solved the wrong problem, however much cleaner
     it looks.

---

## AI decision guidance

When generating code-degradation-diagnosis guidance, keep these principles
in mind:

- **Diagnose before designing** — the requirement-shift/tech-debt
  distinction changes the correct target shape of a refactor, not just
  how the justification is framed.
- **Frame tech debt as a compounded cost, not a character judgment** — this
  is both more accurate (constraints were usually real) and more useful.
- **Use concrete evidence (history, author context) for the diagnosis**,
  and say plainly when it's genuinely unclear rather than forcing a
  verdict.
- **Consulting original authors is about context, not permission** — never
  let this diagnostic step read as gatekeeping who may refactor what.

---

## Success criteria

A strong response should ensure that it:

- **diagnoses requirement-shift vs. tech debt** (or states the diagnosis is
  unclear) before recommending a refactor's target design,
- **designs a requirement-shift fix around the current requirement**, not a
  tidier version of the outdated one,
- **frames tech debt without blame**, naming it as a compounded cost from a
  reasonable-at-the-time trade-off,
- **cites concrete diagnostic evidence** (commit history, author context)
  rather than asserting a diagnosis without support,
- **treats author consultation as a context-gathering step**, never as a
  gate on whether the refactor may proceed.

---

## Example prompts for the AI

- "This code looks like a mess — is it actually tech debt, or did
  something change around it?"
- "Before we redesign this, what can we learn about why it ended up this
  way?"
- "The original author says they were just trying to hit a deadline — does
  that change how we should approach this refactor?"
- "How do we tell whether this needs a redesign or just needs cleaning up?"

---

## Related guidance

Use this tool alongside:

- architecture-refactoring-scope-classification — classify the candidate
  (Local vs. At-Scale) alongside this diagnosis; both shape how much
  process the refactor needs.
- architecture-refactoring-decision-criteria — once a candidate is
  diagnosed, use its trigger/risk framework to decide whether and when to
  actually act on it.
- architecture-refactoring-evidence-gathering — the concrete techniques
  (version-control mining, documentation search) for gathering the
  evidence this diagnosis depends on.
- architecture-third-party-dependency-conflicts — a common concrete cause
  of the "external dependency" requirement-shift subcategory.
