---
name: architecture-third-party-library-selection-checklist

description: Instructs the AI assistant to run a first-impressions check (stability, active maintenance, community adoption, team/backing, documentation) before a deep technical evaluation of a third-party library, to invest proportionally more scrutiny in a framework than a library since a framework is structurally harder to swap out later, to size an abstraction layer to the library's actual vendor-lock-in risk, to treat licensing as a legal question rather than a developer judgment call, to treat security patches as fast-tracked upgrades backed by automated scanning, and ties together the deeper evaluation areas (defaults/concurrency, testability, dependency conflicts) covered by this package's other third-party-library skills.
---

# Third-Party Library Selection Checklist Instructions

When supporting the selection of a third-party library or framework, use this
tool as the first-stop entry point: a quick first-impressions pass before
committing to a deep technical evaluation, and a checklist that ties together
this package's deeper third-party-library skills with the areas they don't
cover — library-vs-framework risk, vendor lock-in, licensing, and security.

---

## Purpose

This tool helps the AI assistant by:

- running a fast, cheap first-impressions check — stability, active
  maintenance, community adoption, the team/organization behind it,
  documentation quality — before investing in the deeper technical
  evaluation this package's other third-party-library skills cover,
- calibrating how much scrutiny a choice deserves by whether it's a library
  (usually cheap to abstract away and later swap) or a framework
  (structurally invasive, and much more expensive to change once adopted),
- sizing an abstraction layer around a dependency to its actual
  deprecation/replacement risk, rather than either abstracting everything
  reflexively or nothing at all,
- treating licensing compatibility as a question for legal review, not a
  judgment call to make unilaterally as a developer,
- treating security vulnerabilities in a dependency as a fast-tracked,
  top-priority upgrade, backed by automated scanning rather than manual
  checking, and distinguishing that urgency from the calmer cadence
  appropriate for routine minor/patch updates.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- checks stability, maintenance activity, community adoption, backing, and
  documentation quality early, before deep technical investigation, and
  treats a weak answer on these as a reason for more caution, not an
  automatic disqualifier — none of them are simple yes/no signals,
- explicitly asks whether a candidate is a library or a framework, and
  applies more thorough up-front investigation to framework choices, since
  they're far harder to walk back later,
- recommends an abstraction layer at a dependency's integration point
  specifically when replacement risk is real, and doesn't recommend one
  reflexively for every dependency regardless of risk,
- flags a restrictive license (e.g., copyleft licenses with source-disclosure
  obligations) as a blocker requiring legal input, never resolves it with a
  developer-level guess,
- treats a disclosed security vulnerability in a dependency as urgent,
  recommends automated vulnerability scanning as the default detection
  mechanism, and separates that urgency from the normal pace of routine
  version upgrades.

---

## Instructions for the AI

1. **Run the first-impressions check before the deep technical dive**
   - **Stability** — does it have a stable release, or is it pre-1.0/beta?
     If pre-stable, is there real confidence it'll stabilize before it's
     needed in production, or is that just hope?
   - **Active maintenance** — is it still being developed, or does it look
     abandoned? A library that solves a narrow, complete problem may
     legitimately need no updates for a long stretch — don't confuse "not
     recently touched because it's genuinely done" with "abandoned," but
     verify which one is actually true rather than assuming the charitable
     answer.
   - **Community adoption** — is it used widely enough that help is easy to
     find? Wide adoption also tends to correlate with more scrutiny having
     already been applied to the code.
   - **Team and backing** — is it maintained by multiple people, ideally
     backed by an organization that depends on it themselves? A solo
     hobby project is a materially different risk than that, even though
     solo projects can and do get diligently maintained for years — treat
     this as a risk factor to weigh, not a disqualifier on its own.
   - **Documentation** — is there real API reference material, conceptual
     documentation, and a usable quick-start, not just inline comments in
     the source?
   - None of these five is a clean yes/no gate — weigh them together, and
     treat a genuinely weak set of answers as a reason to invest more, not
     less, in the deeper evaluation that follows.

2. **Scale scrutiny to whether it's a library or a framework**
   - A library can usually be hidden behind a thin abstraction at the
     integration point without much cost — wrap the calls, and don't let
     anything library-specific leak past that boundary (its exceptions, per
     **architecture-exception-design-and-anti-patterns**; its configuration
     shape, per **architecture-configuration-surface-tradeoffs**). Once
     that's done, swapping the library later is a bounded, local change.
   - A framework is structurally different: it tends to require its own
     constructs throughout the application (inheritance, specific
     annotations/decorators, a required project layout, a lifecycle your
     code has to participate in) rather than being called from one
     integration point. The more of these constructs are threaded through
     the codebase, the more expensive it becomes to ever change frameworks —
     this is usually not a boundary that can be abstracted away cheaply,
     unlike a library's.
   - Because of this asymmetry, apply substantially more up-front
     investigation to a framework choice than to a library choice — the
     cost of getting a library choice wrong is bounded by the abstraction
     around it; the cost of getting a framework choice wrong is closer to
     an application-wide rewrite.

3. **Size the abstraction layer to the dependency's actual replacement risk**
   - Every dependency carries some risk of eventual deprecation or
     replacement — a vendor product changing hands, a cloud service being
     sunset, an open-source library losing traction to a newer alternative
     that solves the same problem more cleanly. This risk isn't uniform
     across dependencies; it's worth actually estimating per dependency,
     not assumed to be either negligible or catastrophic everywhere.
   - Hide a dependency behind an abstraction layer specifically when this
     risk is genuinely high, so a future replacement is a change localized
     to that boundary rather than something that ripples through the
     codebase. Don't recommend an abstraction layer reflexively for every
     dependency regardless of actual risk — that's the same premature
     generality **architecture-flexibility-complexity-tradeoffs** and
     **architecture-simplicity** already argue against, just applied to
     dependency management specifically.
   - Accept that some level of lock-in is unavoidable in practice — not
     every integration point can be cheaply abstracted, and pretending
     otherwise just adds unused complexity. Aim to minimize lock-in where
     the risk is real, not to eliminate it everywhere.

4. **Treat licensing as a legal question, not a developer judgment call**
   - Check a candidate library's license before adopting it, and recognize
     that some licenses carry real obligations — a copyleft license (e.g.
     GPL) can require disclosing the source of code that uses it, which may
     be an outright blocker for a proprietary or internal project.
   - Treat any uncertainty here as a reason to involve legal review, not a
     judgment call to resolve unilaterally as a developer — the cost of
     guessing wrong on licensing can be severe and is a fundamentally
     different kind of risk than a technical mistake.

5. **Fast-track security updates; scan automatically rather than checking manually**
   - Treat a disclosed security vulnerability in a dependency as a
     top-priority, expedited upgrade — the longer an application runs a
     known-vulnerable version, the more time an attacker has to exploit
     it, and the vulnerability's public disclosure itself is what makes the
     exposure window meaningful.
   - Recommend automated dependency-vulnerability scanning (Dependabot,
     Renovate, or an equivalent that can also open the upgrade PR
     automatically) as the default detection mechanism, rather than manually
     checking CVE databases — automation catches this continuously, manual
     checking doesn't.
   - Separate this urgency from routine version-tracking: keep an eye on new
     major versions (investigate the scope of change before committing to
     an upgrade, and be aware of the older version's remaining support
     lifetime) and minor/patch versions (usually safe to adopt quickly under
     semantic versioning — see **architecture-third-party-dependency-conflicts**
     — and worth checking for bugfixes an application might already be
     silently affected by).

---

## AI decision guidance

When generating third-party library selection guidance, keep these principles
in mind:

- **Run the first-impressions check first** — it's cheap, and a weak result
  should raise the bar for everything that follows, not be skipped past.
- **A framework deserves substantially more up-front scrutiny than a
  library** — a library is usually cheap to abstract and later swap; a
  framework structurally isn't.
- **Abstraction layers should be sized to actual replacement risk**, not
  applied reflexively to every dependency regardless of risk.
- **Licensing uncertainty goes to legal review, never a developer's own
  guess.**
- **Security vulnerabilities get fast-tracked, automated-scanning-driven
  upgrades** — a different urgency and mechanism than routine version
  tracking.

---

## Success criteria

A strong response should ensure that it:

- **checks stability, maintenance, adoption, backing, and documentation**
  before recommending deep technical investment in a candidate,
- **applies more scrutiny to a framework than to a library**, explaining why
  the two carry structurally different switching costs,
- **recommends an abstraction layer proportional to actual replacement
  risk**, not as a reflexive default,
- **routes licensing uncertainty to legal review** rather than resolving it
  informally,
- **treats security patches as urgent and automation-driven**, distinct from
  routine version upgrades.

---

## Example prompts for the AI

- "We're considering a fairly new library with no stable release yet — how
  should that factor into the decision?"
- "Is it worth building an abstraction layer around this dependency, or is
  that overkill?"
- "This library is GPL-licensed — can we use it in our proprietary
  codebase?"
- "How urgently do we need to respond to a CVE in one of our dependencies?"

---

## Related guidance

Use this tool alongside:

- architecture-third-party-defaults-and-concurrency — the deeper evaluation
  of a library's configuration defaults and concurrency model.
- architecture-third-party-testability-evaluation — the deeper evaluation of
  a library's testability.
- architecture-third-party-dependency-conflicts — the deeper evaluation of a
  library's transitive dependencies, versioning, and dependency footprint.
- architecture-exception-design-and-anti-patterns /
  architecture-configuration-surface-tradeoffs — what "don't leak
  library-specific details past the abstraction boundary" means concretely
  for exceptions and configuration, referenced in point 2.
- architecture-flexibility-complexity-tradeoffs / architecture-simplicity —
  the general premature-generality argument behind point 3's "don't
  abstract reflexively."
- architecture-library-api-evolution — the same evaluation criteria (health,
  versioning policy, responsiveness) applied from the other side, when
  you're the one publishing the library others will evaluate this way.
- architecture-trend-adoption-discipline — the earlier gate that decides
  whether this checklist's deep evaluation is even warranted: confirming the
  problem a trendy framework claims to solve actually exists before
  investing in evaluating candidates for it.
