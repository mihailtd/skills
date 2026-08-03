---
name: architecture-library-api-evolution

description: Instructs the AI assistant on practical library-versioning discipline beyond SemVer's bare rules — keeping the public API surface minimal by default since an accidentally-exported name becomes a permanent liability while an internal one is a free rename, evaluating whether a change is actually breaking against realistic consumer code rather than defaulting to "when in doubt, assume breaking" (expensive, not cautious), deprecating before removing via a bridging release that ships the new shape alongside the old one first, batching breaking changes into infrequent well-documented major versions, and applying the same discipline with more flexibility to internal-only libraries where usage is often actually auditable.
---

# Library API Evolution Instructions

When supporting the ongoing evolution of a library or internal package — not
just picking a version number for one change, but the surrounding discipline
that makes future changes cheaper — use this tool alongside
**architecture-semantic-versioning** for the compatibility-rules layer this
skill builds practical habits on top of.

---

## Purpose

This tool helps the AI assistant by:

- treating a minimal public API surface as a deliberate default, not an
  afterthought — every public name is a future compatibility liability, and
  an internal one costs nothing to fix or rename later,
- evaluating whether a change is actually breaking against realistic
  consumer code, not a worst-case hypothetical, and correcting the instinct
  that "when in doubt, assume breaking" is the cautious choice — it's
  actually the expensive one, since it forces a major-version bump and
  everything that costs downstream,
- recommending a bridging release — shipping the new API shape alongside
  the deprecated old one in a minor version, before removing the old shape
  in the next major version — as the default way to reduce the pain of an
  eventual breaking change,
- recommending that breaking changes be batched into infrequent, thoroughly
  documented major versions rather than trickled out one bump at a time,
- applying this same discipline, with appropriately more flexibility, to
  internal-only libraries — where usage is often genuinely auditable, unlike
  an open-source library with unknown consumers, but that doesn't mean no
  strategy is needed.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- makes new library surface internal/private by default, requiring a
  deliberate decision to make something public, rather than defaulting to
  public and hoping nothing needs to change later,
- assesses a candidate change against what realistic consumer code would
  actually do, and is willing to ship a technically-breaking-in-theory
  change as a minor version when there's real evidence it won't affect
  actual consumers — transparently documented, not quietly slipped in,
- introduces a new API shape as an addition alongside the old one (marked
  deprecated) in a minor release, and only removes the old shape in a
  subsequent major release, rather than replacing in place,
- keeps a running list of intended breaking changes and batches them into
  infrequent major versions with real migration documentation, rather than
  bumping major on every individual breaking change,
- picks a deliberate versioning approach for internal-only libraries too —
  "live at head," independently versioned internal packages, or a hybrid —
  and prefers gradual migration (shrinking usage of an old shape before
  removing it) over an uncoordinated break, unless a genuinely cheaper
  coordinated change with acceptable downside is available.

---

## Instructions for the AI

1. **Default to a minimal public API surface**
   - Every public class, function, or field is a future compatibility
     liability — even a naming mistake (a typo in a public name) becomes a
     breaking change to fix once it's shipped, while the identical mistake
     in an internal name costs nothing to correct. Keep new surface private
     by default; making something public should be a deliberate decision,
     not the default outcome of not thinking about it.
   - Prefer shipping an intentionally restrictive surface first — the
     essential functionality plus sensible defaults — and let real user
     feedback reveal what additional flexibility is actually needed, rather
     than speculatively designing in flexibility nobody has asked for yet.
     This is the same premature-generality trap covered generally in
     **architecture-flexibility-complexity-tradeoffs** and
     **architecture-simplicity**, applied specifically to a library's public
     surface.
   - Where genuine inheritance/subclassing is intended as part of the public
     contract, document that explicitly — an overridable method that another
     method calls is no longer a private implementation detail once
     documented as overridable, it's part of the contract. This repository's
     functional-lite default (composition over inheritance — see
     **architecture-inheritance-coupling-tradeoffs**) avoids most of this
     exposure surface in the first place, which is one more reason to prefer
     it.

2. **Evaluate "breaking" against realistic consumer code, not a worst-case hypothetical**
   - When assessing whether a change is breaking, construct the most
     *likely* consumer code that could be affected, not the most obscure
     theoretically-possible one. A change that only breaks code relying on
     some highly unusual language feature or edge case is a reasonable
     candidate for a minor bump, documented honestly, rather than an
     automatic major bump.
   - Correct the instinct that "when in doubt, assume it's breaking" is the
     safe, cautious choice — it isn't. Every unnecessary major version
     propagates cost through every consumer and everything that depends on
     them (see **architecture-third-party-dependency-conflicts** for how
     that propagation actually plays out). If there's solid evidence a
     technically-breaking change won't affect real consumers, shipping it as
     a minor version can be the right call — but do so transparently,
     documenting the reasoning, not silently deviating from semantic
     versioning's rules without explanation.
   - Document what your library or team considers a breaking change, since
     this always involves judgment calls that no tool fully automates —
     automated compatibility checkers encode their own judgment calls (e.g.
     treating a new public class as automatically non-breaking, even though
     it can cause a naming collision) and can't detect semantic breaking
     changes at all (see **architecture-semantic-versioning**'s Hyrum's Law
     guidance). A documented policy avoids surprising consumers with
     inconsistent classification later.
   - Treat a deprecation warning itself as a courtesy, not a breaking
     change — it's advance notice of an eventual change, which is exactly
     what versioning exists to communicate. Consumers who've configured
     "treat warnings as errors" have made an active choice to be alerted
     early; that's a reasonable tradeoff they opted into, not a bug in the
     library's versioning.

3. **Deprecate before removing: ship the new shape first, in a bridging release**
   - When an API shape needs to change (a wrong name, a wrong parameter
     type, a fundamentally different design), don't just replace it in the
     next major version with no warning. Ship the new shape *alongside* the
     old one in a minor release first: add the new function/method/parameter,
     and mark the old one deprecated (in Python, `warnings.deprecated`
     — PEP 702, stdlib since 3.13 — or `typing_extensions.deprecated` for
     broader version support) so it still works exactly as before, just with
     a visible signal that it's going away.
   - This gives consumers a full release cycle to migrate at their own pace
     — ignoring the warning costs them nothing immediately, but they have
     concrete, actionable notice of exactly what to change and when. Only
     remove the deprecated shape in the *next* major version, after it's had
     time to be seen and acted on.
   - This converts what would otherwise be an abrupt, all-at-once breaking
     change into a two-step migration: a free, warning-only step now, and a
     scoped, well-telegraphed removal later — substantially cheaper for
     consumers than a surprise removal, for a modest amount of extra
     maintenance (supporting both shapes for one release cycle).

4. **Batch breaking changes into infrequent, well-documented major versions**
   - Keep a running list of changes intended to be breaking as they're
     identified, rather than shipping a new major version for each one
     individually — the cost of a major version (see point 2) is largely
     independent of how many breaking changes it bundles, so bundling
     amortizes that cost across more value delivered per bump.
   - When a major version does ship, document every breaking change it
     contains — including subtle, easy-to-miss ones — and provide a
     migration guide, not just a changelog entry. The goal is that a
     consumer can read one document and know exactly what to check and
     change, rather than having to diff behavior themselves.

5. **Apply the same discipline, with more flexibility, to internal-only libraries**
   - An internal library (no external consumers, code that never leaves your
     own systems) genuinely does have more room to maneuver — usage is
     often actually auditable (you can search your own codebase for every
     call site), which an open-source library's maintainer usually can't do.
     That doesn't mean no versioning strategy is needed — pick one
     deliberately: "live at head" (everything always builds against the
     latest version, no independent version numbers), independently
     versioned internal packages, or a hybrid, and make sure everyone who
     depends on the library understands which one is in effect and what it
     implies for how changes propagate.
   - Prefer a gradual migration over an uncoordinated break by default: add
     the new shape, migrate consumers to it incrementally, shrink usage of
     the old shape, and only remove it once usage has genuinely dropped to
     zero — ideally with a cooling-off period afterward in case a recent
     change needs to be rolled back. Treat an all-at-once coordinated break
     as the exception, reasonable only when the gradual approach costs more
     than it's worth and a brief, accepted disruption is genuinely cheaper —
     the same tradeoff that shows up in accepting brief downtime for a data
     schema migration rather than engineering a fully online one.

---

## AI decision guidance

When generating library-evolution guidance, keep these principles in mind:

- **Public API surface is a liability, not a convenience** — default to
  internal, and require a deliberate decision to make something public.
- **"When in doubt, assume breaking" is the expensive choice, not the
  cautious one** — evaluate against realistic consumer code, and ship
  transparently-documented exceptions to strict SemVer when there's real
  evidence they're safe.
- **Deprecate before removing, via a bridging release** — the new shape
  ships first, alongside the old one, so migration is optional and free
  before it's ever mandatory.
- **Breaking changes should be batched into infrequent, thoroughly
  documented major versions**, not trickled out one bump at a time.
- **Internal-only libraries still need a deliberate versioning strategy**,
  just one that can lean on gradual migration more heavily since usage is
  often genuinely auditable.

---

## Success criteria

A strong response should ensure that it:

- **keeps new API surface internal by default**, requiring a deliberate
  choice to expose it publicly,
- **evaluates breaking-ness against realistic consumer code**, and is
  willing to make (and document) a judgment call rather than reflexively
  bumping major,
- **recommends a bridging release** (new shape added, old shape deprecated)
  before any planned removal,
- **recommends batching breaking changes** into major versions with real
  migration documentation,
- **picks a deliberate versioning strategy for internal libraries too**, and
  defaults to gradual migration over an uncoordinated break.

---

## Example prompts for the AI

- "Should this new helper function be public or kept internal?"
- "Is this change actually breaking, or are we being overly cautious?"
- "How do we rename this method without immediately breaking everyone using
  it?"
- "We don't publish this library externally — does it even need a
  versioning strategy?"

---

## Related guidance

Use this tool alongside:

- architecture-semantic-versioning — the compatibility-rule layer (SemVer,
  Hyrum's Law) this skill's practical habits build on top of.
- architecture-third-party-dependency-conflicts — how a breaking change
  propagates through a dependency graph once it ships, from the consumer's
  perspective.
- architecture-third-party-library-selection-checklist — the same
  first-impressions evaluation (stability, maintenance, versioning policy,
  responsiveness) applied when *you* are the one selecting a dependency to
  build a library on top of.
- architecture-inheritance-coupling-tradeoffs — why this repository's
  composition-over-inheritance default avoids much of the "overridable
  method becomes part of the public contract" exposure this skill's point 1
  describes.
- architecture-flexibility-complexity-tradeoffs / architecture-simplicity —
  the general premature-generality argument behind shipping a minimal public
  surface first.
