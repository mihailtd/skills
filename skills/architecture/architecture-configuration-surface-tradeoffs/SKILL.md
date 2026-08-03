---
name: architecture-configuration-surface-tradeoffs

description: Instructs the AI assistant to treat a dependency's configuration surface as a design decision with the same flexibility-vs-maintenance tradeoff as the rest of an API — passing a dependency's raw configuration straight through costs nothing today but turns every future breaking change in that dependency into an unbuffered breaking change for every one of your own users, while owning and mapping a narrow, purpose-specific configuration section costs ongoing mapping work but insulates users from the dependency's evolution — and rejects the "silently rewrite the user's config file for them" workaround that a passthrough design tends to produce once it meets a breaking change.
---

# Configuration Surface Tradeoffs Instructions

When supporting the design of a tool or service that wraps a third-party
dependency, use this tool to evaluate whether the dependency's configuration
should be passed through directly or owned and mapped internally — a decision
with the same shape as any other flexibility-vs-maintenance tradeoff, but one
that's easy to make by accident rather than deliberately.

---

## Purpose

This tool helps the AI assistant by:

- treating a dependency's configuration surface as part of your own API's
  design, not an implementation detail to decide casually — a
  **passthrough** approach (forwarding the dependency's own configuration
  shape to your callers unmodified) and an **ownership** approach (exposing
  only your own purpose-specific settings, mapped internally onto the
  dependency's configuration) are both deliberate choices with different
  cost profiles,
- showing that the cost of each approach depends on how the dependency's
  configuration actually changes: passthrough is nearly free for an additive
  change (a new optional setting) but has no buffer at all for a breaking
  change (a setting renamed, removed, or deprecated) — the dependency's
  breaking change becomes your tool's breaking change, simultaneously, for
  every caller,
- rejecting, as a genuine anti-pattern rather than an acceptable workaround,
  the runtime config-mutation hack that a passthrough design tends to
  produce once it meets a breaking dependency change — silently rewriting or
  patching a caller's configuration to paper over the change,
- tying the choice to who a tool's actual callers are and whether you
  control their release cadence — many independent, external callers favor
  ownership despite its ongoing cost; a small number of internal, coordinated
  callers can reasonably tolerate passthrough's occasional breaking change.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- identifies explicitly whether a given integration point is passthrough or
  ownership, rather than drifting into passthrough by default because
  forwarding "the rest of the config" to a dependency's builder was the
  easiest thing to write,
- evaluates the tradeoff against both additive and breaking dependency
  changes — not just the common, cheaper-looking additive case — since the
  breaking case is where passthrough's cost asymmetry actually bites,
- never recommends silently rewriting or patching a caller's configuration
  at runtime to hide a breaking passthrough change — surfaces it as an
  explicit, version-bumped breaking change in your own tool instead,
- recommends ownership specifically when a tool has many independent
  external callers whose release cadence isn't controlled, and accepts
  passthrough as a reasonable deliberate choice for a small, coordinated set
  of internal callers,
- accounts for the ownership approach's cost multiplying across every
  independent tool or service that separately wraps the same dependency.

---

## Instructions for the AI

1. **Name which approach a given integration point actually uses**
   - **Passthrough:** the dependency's own configuration format (or a
     section of it) reaches your tool's callers verbatim — your tool
     forwards it to the dependency without interpreting or reshaping it.
     Zero mapping code, but your tool's own configuration contract is,
     silently, whatever the dependency's contract happens to be today.
     ```yaml
     # caller's config file — the `auth` section is the dependency's own shape,
     # forwarded to it unmodified
     auth:
       strategy: username-password
       username: user
       password: pass
     batch:
       size: 100
     ```
   - **Ownership:** your tool defines its own, narrower configuration
     surface, and internally maps each of its own settings onto whatever the
     dependency actually needs.
     ```yaml
     # caller's config file — only the tool's own settings, dependency shape
     # never visible to the caller
     streaming:
       username: u
       password: p
       maxTimeMs: 100
     ```
   - Before recommending either, identify explicitly which one a design is
     doing — it's easy to drift into passthrough without treating it as a
     deliberate choice, simply by forwarding an unrecognized config section
     straight to a dependency's builder.

2. **Weigh the two approaches against how the dependency actually changes**
   - For an **additive** change (a new optional setting with a sensible
     default), passthrough is nearly free — the new setting flows through
     untouched, and only callers who want it need to add it. Ownership pays a
     real cost here: every new dependency setting needs a corresponding
     addition to your own configuration surface and mapping code, multiplied
     by however many independent tools wrap the same dependency.
   - For a **breaking** change (a setting renamed, removed, or deprecated),
     the cost profile inverts sharply: passthrough has no buffer at all — the
     dependency's breaking change becomes your tool's breaking change,
     simultaneously, for every caller, the moment they upgrade. Ownership
     absorbs the change inside the mapping code — update the mapping once,
     and callers who never knew about the dependency's internals don't need
     to know it changed at all.
   - Because breaking changes are rarer than additive ones but far more
     damaging when they land on unbuffered callers, don't judge the tradeoff
     only by the additive case — explicitly reason about what a breaking
     dependency change would do to your callers before choosing passthrough.

3. **Reject config-mutation workarounds as an anti-pattern, not a fix**
   - Once a passthrough design meets a breaking dependency change, the
     tempting "fix" is to intercept the caller's configuration at runtime and
     silently rewrite it into the new shape the dependency now expects
     (renaming a field, replacing a value, even writing a patched copy back
     to disk) before forwarding it. Reject this outright rather than
     refining it.
   - This is dangerous for concrete reasons beyond "it's hacky": it operates
     on the caller's own data without their knowledge or consent (rewriting a
     plaintext credential into a hashed one on their behalf is a
     representative real example), it's brittle against any further change
     in either the dependency's or your own assumed structure, and it hides
     the actual breaking change from the caller instead of surfacing it as a
     version bump they consciously opt into.
   - It also introduces exactly the coupling passthrough was supposedly
     avoiding: a component whose only job was loading/forwarding
     configuration sections now has to know the dependency's exact internal
     setting names well enough to rewrite them — a config loader and a
     third-party client's configuration schema become tightly coupled to
     each other for no reason beyond patching over the workaround.
   - The honest options once a passthrough design meets a breaking dependency
     change: pass the breaking change straight through as a breaking change
     in your own tool (a major version bump, clearly documented and
     communicated), or move to the ownership/mapping approach going forward.
     Ownership has a genuine third option a passthrough design doesn't:
     because the mapping layer already sits between callers and the
     dependency, it can support both the old and new caller-facing shapes in
     parallel for a migration window (e.g. accepting either a plain-text
     password to hash internally, or a new field carrying an
     already-hashed one directly) — callers migrate at their own pace
     instead of everyone breaking, or being silently patched, at once. This
     is the concrete payoff ownership buys with its ongoing mapping cost.

4. **Choose based on who the callers are and who controls their release cadence**
   - Favor **ownership** when a tool has many independent, external callers
     whose release cadence you don't control — insulating them from the
     dependency's churn is worth the ongoing mapping cost, and the cost of a
     passthrough breaking change (simultaneous, unbuffered breakage for
     every caller) is proportionally larger the more callers there are.
   - Passthrough can be a reasonable, deliberate choice for a small number of
     internal, coordinated callers who can absorb an occasional breaking
     change directly — especially when the dependency exposes many settings
     and the ownership/mapping cost would be substantial relative to the
     actual risk.
   - Multiply the ownership approach's maintenance cost by the number of
     independent tools/services that separately wrap the same dependency —
     each one pays its own per-setting mapping cost, so the case for a
     shared abstraction (rather than N separate ownership-style wrappers)
     strengthens as that count grows (see
     **architecture-code-duplication-tradeoffs** for the extraction decision
     itself once that's the case).
   - Factor in how much control you actually have over the dependency's own
     evolution, not just how many callers you have. If you own or can
     meaningfully influence the dependency's release process (an internal
     library, or a vendor relationship where you have real input), you may
     be able to prevent or minimize breaking changes at the source — which
     weakens the case for paying the ownership cost, since the scenario
     ownership protects against becomes rare by design. If the dependency
     evolves unpredictably and outside your influence (a third-party SaaS
     SDK, an external open-source project on its own roadmap), assume
     breaking changes will happen on their schedule, not yours, and weight
     the decision toward ownership accordingly.

---

## AI decision guidance

When generating configuration-design guidance, keep these principles in mind:

- **A dependency's configuration surface is part of your own API's design,
  not an implementation detail** — decide deliberately between passthrough
  and ownership rather than defaulting to whichever was easiest to wire up
  first.
- **Passthrough is cheap for additive dependency changes and dangerous for
  breaking ones** — evaluate the tradeoff against the breaking case
  specifically, since that's where the cost asymmetry actually bites.
- **Never recommend silently rewriting or patching a caller's configuration**
  to paper over a breaking passthrough change — surface it as an explicit
  breaking change in your own tool, or invest in ownership going forward.
- **Ownership's real payoff is a gradual migration path** — because its
  mapping layer already sits between callers and the dependency, it can
  support old and new caller-facing shapes side by side during a transition,
  something a passthrough design structurally cannot offer.
- **The more independent callers a tool has, and the less control you have
  over both their release cadence and the dependency's own evolution, the
  stronger the case for ownership** despite its ongoing mapping cost.

---

## Success criteria

A strong response should ensure that it:

- **correctly identifies whether a design is passthrough or ownership**, and
  names the specific cost each pays,
- **evaluates the tradeoff against both additive and breaking dependency
  changes**, not just the additive case,
- **rejects runtime config-mutation workarounds outright** when a breaking
  passthrough scenario comes up, recommending an explicit breaking change or
  a move to ownership instead,
- **ties the ownership-vs-passthrough recommendation to the number and
  coordination-level of the tool's actual callers, and to how much control
  you have over the dependency's own release process**,
- **offers a dual-path migration (supporting both the old and new
  caller-facing shapes for a transition window) as ownership's constructive
  alternative** to a forced simultaneous break, when that's the scenario in
  play.

---

## Example prompts for the AI

- "Should our CLI tool just forward the underlying SDK's config file, or
  define its own settings and map them?"
- "The library we depend on is deprecating a config option — how do we roll
  that out without breaking every user at once?"
- "Someone suggested we auto-migrate users' config files at runtime when we
  detect the old format — is that a good idea?"
- "We have five internal services that all wrap the same client library's
  config — should each one own its mapping, or should we share one?"

---

## Related guidance

Use this tool alongside:

- architecture-flexibility-complexity-tradeoffs — the general
  abstraction-with-a-default-implementation pattern this skill's ownership
  approach specializes for configuration specifically.
- architecture-code-duplication-tradeoffs — the cost-scaling argument in
  point 4 (multiply by N services) and the shared-library extraction option
  once several independent tools separately own the same mapping.
- architecture-inheritance-coupling-tradeoffs
- architecture-simplicity
