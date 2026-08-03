---
name: architecture-semantic-versioning

description: Instructs the AI assistant on semantic versioning's precise rules (major.minor.patch compatibility guarantees, prerelease labels, why 0.x.y should be reserved for early prototyping rather than an entire pre-1.0 development phase) and on Hyrum's Law — with enough users, every observable behavior gets depended on regardless of what the contract promises, so a version number communicates intent, not a guarantee, and three specific categories of change (validation loosening/tightening, exposed implementation details, performance characteristics) look backward compatible but frequently aren't.
---

# Semantic Versioning and Hyrum's Law Instructions

When supporting decisions about how to version a library, package, or public
API, use this tool for two things: applying semantic versioning's actual rules
precisely (not just "bump the number that feels right"), and recognizing where
a version number's promise runs out — Hyrum's Law is why even a strictly
SemVer-compliant change can still break real consumers.

---

## Purpose

This tool helps the AI assistant by:

- applying semantic versioning's precise compatibility rules — what a
  major/minor/patch bump each actually promises — rather than treating
  version numbers as an arbitrary increasing counter,
- recommending prerelease labels (`1.0.0-beta.1`) over a `0.x.y` version
  range for anything beyond the very earliest prototyping, since `0.x.y`
  only cleanly covers the run-up to the *first* stable release and breaks
  down as a scheme the moment a second major version is needed,
- explaining Hyrum's Law as the reason a version number is a communication
  of *intent*, not an enforceable guarantee — with enough consumers, every
  observable behavior becomes something somebody depends on, whether or not
  it was ever part of the documented contract,
- naming three specific categories of change that routinely look backward
  compatible but silently aren't: loosening or tightening input validation,
  changes that expose what used to be an internal implementation detail, and
  changes to performance characteristics that are themselves an observable
  behavior.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- picks a version bump (major/minor/patch) based on semantic versioning's
  actual compatibility rules for the change being made, not on how large the
  change feels or how much code it touched,
- uses prerelease labels for pre-stable releases from the first public
  release onward, reserving a `0.x.y` major version for private,
  very-early-stage prototyping that will never be depended on externally,
- distinguishes a marketing/product version (meant to be memorable, not
  precise) from a technical/package version (meant to communicate
  compatibility precisely) when both exist, and doesn't conflate the two,
- treats "this doesn't change the documented public contract" as
  insufficient justification for calling a change non-breaking — checks
  specifically for validation changes, newly-observable implementation
  details, and performance-characteristic changes before declaring
  something safe,
- accepts that no versioning scheme can make every change perfectly safe for
  every consumer, and focuses instead on making the *intent* of a change
  clear and giving consumers the information they need to assess their own
  risk.

---

## Instructions for the AI

1. **Apply semantic versioning's compatibility rules precisely**
   - A version is `major.minor.patch`. The rules, for the same entity across
     two versions:
     - **Different major numbers** — no compatibility guarantee at all.
       Anything can have changed.
     - **Same major, different minor** — the higher minor version must be
       backward compatible with the lower one (new, non-breaking
       functionality was added).
     - **Same major and minor, different patch** — must be compatible in
       *both* directions (patches are for bug fixes only, not new
       functionality) — with one caveat: if consumer code depended on the
       buggy behavior a patch fixes, upgrading can still break it. That's a
       real, known limitation of the patch guarantee, not a violation of
       it.
   - Translate this into a concrete consumer-facing rule of thumb: a major
     bump means "budget real time and testing for this upgrade"; a minor
     bump means "should be safe, but is adding something new"; a patch bump
     means "should be safe to take immediately," with the bug-fix caveat
     above.

2. **Use prerelease labels, not `0.x.y`, for anything beyond very early prototyping**
   - `0.x.y` versions have no compatibility guarantees between any two of
     them — `0.2.0` can be entirely incompatible with `0.1.0`. This scheme
     only cleanly covers the lead-up to a *first* stable `1.0.0` release; it
     doesn't generalize to a second major version's own prerelease phase
     (`1.8`, `1.9`, `2.0` for a second major version violates SemVer, since a
     major bump is supposed to signal breaking changes, not a version
     sequence leading into one).
   - Prerelease labels (`1.0.0-alpha.1`, `1.0.0-beta.1`, `2.0.0-alpha.1`)
     solve this cleanly and are consistent across every major version's own
     prerelease sequence — recommend them from the first public release
     onward. Reserve `0.x.y` for genuinely early, private prototyping that
     will never be published for external consumers to depend on.

3. **Keep marketing versions and technical versions separate concepts**
   - A marketing/product version exists to be memorable and to sell — it
     has no obligation to convey compatibility information, and often
     doesn't correspond to the technical version at all (a product's
     marketing version might reset to a lower number for a new major
     release while its technical version keeps climbing).
   - A technical/package version (what SemVer governs) exists specifically
     to convey compatibility information to tooling and other developers.
     When both exist for the same thing, don't let the marketing version's
     simplicity create false expectations about the technical version's
     compatibility guarantees, or vice versa.

4. **Apply Hyrum's Law: a contract's promises are not the same as what consumers actually depend on**
   - Hyrum's Law: "With a sufficient number of users of an API, it does not
     matter what you promise in the contract: all observable behaviors of
     your system will be depended on by somebody." This isn't a reason to
     never change anything — it's a reason to actively look for the
     specific categories of change most likely to violate it, rather than
     assuming "not in the documented contract" means "safe to change."
   - **Validation changes** — loosening a previously-strict validation
     (accepting a value that used to be rejected) can silently change
     downstream behavior for consumers who were implicitly relying on the
     rejection (e.g., a null/empty check that used to fail fast now lets a
     null propagate somewhere it's not handled). Tightening validation (now
     rejecting something that used to be accepted) is more obviously a
     breaking change, but is easy to mis-classify as "just a bug fix."
     Consider whether an additive path (a new optional parameter, a new
     endpoint variant) avoids the silent-break risk instead of changing the
     existing one's behavior in place.
   - **Newly-observable implementation details** — a change that alters
     which internal piece of code actually does the work (which function
     an object delegates to internally, the internal order operations
     happen in) can be invisible in the source-level contract but change
     behavior for a consumer that happened to depend on the old internal
     path — most commonly via subclassing/overriding a method that used to
     be a pure implementation detail. This repository's functional-lite
     style (composition over inheritance — see
     **architecture-inheritance-coupling-tradeoffs**) structurally avoids
     the specific mechanism this category describes in the source book
     (overridable methods), but the general principle still applies to
     anything callers can hook into or observe beyond the documented
     return value.
   - **Performance-characteristic changes** — a change that makes one usage
     pattern faster at the cost of making a different, previously-efficient
     pattern slower is an observable behavior change even though inputs and
     outputs stay identical. For a library or API where performance is part
     of what consumers reasonably depend on, treat a meaningful shift in
     performance characteristics as a compatibility concern worth calling
     out explicitly, not just an internal optimization.
   - The practical takeaway: before declaring a change non-breaking because
     it doesn't touch the documented public contract, check it against
     these three categories specifically — they're where "technically
     compatible" and "actually safe to ship" most often diverge.

---

## AI decision guidance

When generating versioning guidance, keep these principles in mind:

- **A version bump's size should follow SemVer's actual compatibility
  rules**, not a subjective sense of how big the change feels.
- **Prerelease labels, not `0.x.y`, are the right default for pre-stable
  releases** beyond the earliest private prototyping — `0.x.y` doesn't
  generalize past the first major version.
- **Marketing and technical versions serve different purposes** — don't
  let one create false expectations about the other.
- **"Not in the documented contract" doesn't mean "safe to change"** —
  Hyrum's Law means real consumers depend on undocumented behavior too;
  check validation changes, newly-observable implementation details, and
  performance-characteristic shifts specifically before calling a change
  non-breaking.

---

## Success criteria

A strong response should ensure that it:

- **assigns version bumps according to SemVer's actual major/minor/patch
  rules**, including the patch-level bug-fix caveat,
- **recommends prerelease labels over an extended `0.x.y` range** for
  pre-stable software beyond early private prototyping,
- **keeps marketing and technical versioning concepts distinct** when both
  are relevant,
- **checks a proposed "non-breaking" change against validation changes,
  exposed implementation details, and performance-characteristic shifts**
  before accepting that classification.

---

## Example prompts for the AI

- "Should this change be a minor or a patch release?"
- "We're about to make our first public release — should we start at
  0.1.0 or use a prerelease label?"
- "We loosened a validation rule to accept more input — is that a breaking
  change?"
- "Is it actually safe to call this change backward compatible just because
  the public API signature didn't change?"

---

## Related guidance

Use this tool alongside:

- architecture-third-party-dependency-conflicts — the practical consequence
  of SemVer's major-version-boundary rule: how a version conflict between
  two required versions of the same dependency actually gets resolved (or
  fails to).
- architecture-inheritance-coupling-tradeoffs — the composition-over-inheritance
  default that avoids the specific "exposed implementation detail via an
  overridable method" failure mode this skill's point 4 describes.
- architecture-configuration-surface-tradeoffs — the same
  contract-vs-actual-dependency tension (Hyrum's Law, essentially) applied
  specifically to a dependency's configuration surface rather than its code
  API.
- architecture-library-api-evolution — the practical habits (minimal public
  surface, bridging releases, batched major versions) built on top of this
  skill's compatibility rules.
- architecture-network-api-versioning — the same versioning concerns applied
  to a network API instead of a library, where multiple versions coexist
  continuously rather than sequentially.
- architecture-data-storage-schema-evolution — versioning concerns one layer
  further down, for the storage schema underneath a library or API, where
  "breaking" is defined by the specific storage technology rather than by
  SemVer's rules.
