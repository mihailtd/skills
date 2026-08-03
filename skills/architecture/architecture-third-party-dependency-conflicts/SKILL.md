---
name: architecture-third-party-dependency-conflicts

description: Instructs the AI assistant to never depend directly on a third-party library's transitive dependencies (that's an implementation detail the library owner is free to change without warning) — add an explicit, direct dependency instead when the application genuinely needs the same capability, recognize the diamond-dependency shape and understand how a version conflict actually manifests (uv fails loudly at lock time rather than silently resolving one, a direct consequence of Python's shared-only import model having no isolated-dependency escape hatch the way some ecosystems do), and prefer libraries with a small, minimal dependency footprint since every dependency they bring in becomes part of the application's own dependency graph and its container image size.
---

# Third-Party Dependency Conflict Instructions

When supporting library selection or dependency management, use this tool to
treat every dependency a third-party library brings in as something the
calling application now inherits, whether it asked for it or not — and to
avoid depending on those inherited, transitive dependencies directly, since
they're an implementation detail the library owner can change without
warning.

---

## Purpose

This tool helps the AI assistant by:

- treating "this library's transitive dependency happens to provide a
  capability my code also needs" as a trap, not a convenience — the
  transitive dependency isn't part of the library's public contract, and the
  library owner can swap it, upgrade it incompatibly, or drop it entirely
  without that being a breaking change from *their* perspective,
- recommending an explicit, direct dependency on a capability the
  application genuinely needs, even when a library already pulls it in
  transitively,
- explaining that adding a direct dependency doesn't eliminate the
  conflict risk, it relocates it: two different declared or transitive
  requirements on different versions of the same library can still collide,
  and the build/package tool resolves that collision silently by picking
  one, which can produce a failure with no obvious connection to its actual
  cause,
- weighing a library's own dependency footprint (how many further
  dependencies it drags in) as a real factor in choosing between
  alternatives, not an afterthought to the library's own functionality.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- never imports or calls a class/module/package that arrived only as a
  transitive dependency of something else — if the application genuinely
  needs that capability, it gets its own direct, declared dependency on it,
- understands and can explain why relying on a transitive dependency creates
  coupling to an implementation detail: a future version of the library that
  no longer needs that dependency (or needs an incompatible version of it)
  breaks the application with no warning, since removing an undeclared,
  transitively-available class isn't a breaking change from the library
  owner's point of view,
- recognizes that adding a direct dependency is the right fix but doesn't
  make version conflicts impossible — two different requirements on
  different versions of the same library are still a real, if less common,
  failure mode, and it manifests as a runtime error (a missing method, a
  changed signature) rather than a clear message pointing at the actual
  cause,
- factors a candidate library's own dependency footprint into a selection
  decision between alternatives, preferring fewer and more minimal
  dependencies when other factors are comparable.

---

## Instructions for the AI

1. **Never depend directly on a library's transitive dependencies**
   - When a library (say, an HTTP client) happens to also pull in a
     capability the application separately needs (say, JSON processing),
     resist the shortcut of just importing that capability from wherever the
     transitive dependency makes it available. It's tempting specifically
     because it requires zero extra configuration — the classes are already
     on the classpath/available in the environment — but that convenience is
     exactly the trap.
   - The transitive dependency is an implementation detail of the library
     that brought it in, not a commitment to the application. The library's
     own next release is free to change, upgrade incompatibly, or drop that
     dependency entirely, since — from the library owner's perspective —
     it was never part of their public API in the first place. When that
     happens, the application's code silently stops compiling or working,
     for a reason that has nothing to do with any change the application's
     own team made.

2. **Add a direct, explicit dependency when the application genuinely needs the capability**
   - If application code needs JSON processing, logging, or any other
     capability that a dependency also happens to use internally, declare a
     direct dependency on it rather than borrowing the transitive one. This
     makes the application's own actual requirements explicit and keeps them
     independent of whatever internal choices the other library makes over
     time.
   - This is the correct fix, but be explicit that it doesn't eliminate risk
     entirely — it trades an *invisible* coupling (silently broken by a
     library update with no warning) for a *visible, resolvable* one (an
     explicit version requirement that a build/package tool can reason
     about and that shows up in a dependency tree or lockfile).

3. **Understand how a version conflict actually manifests once you have two requirements on the same library**
   - This is called a **diamond dependency**: your application depends on
     library A and library B directly, and both A and B themselves depend
     on library C — but potentially different, incompatible versions of C.
     Drawn as a graph, the shape (application → A → C, application → B → C)
     is a diamond, and it's the shape that matters, not the specific
     libraries — any dependency two or more of your other dependencies
     share can end up in this position.
   - Once both the application and one of its dependencies require the same
     underlying library, but at different versions, a build or package tool
     has to resolve that to a single version actually used at runtime.
     Whether that resolution is silent or loud depends on the ecosystem —
     don't assume the JVM's classpath behavior (pick one, defer the failure
     to a confusing runtime error) generalizes everywhere. It also depends
     on whether the ecosystem supports *isolated* dependencies (each
     consumer gets its own private copy, so two different versions of the
     same library can coexist) or only *shared* ones (the whole process
     uses exactly one copy) — Python's import system is shared-only in
     normal operation, with no equivalent of per-consumer classloader
     isolation, so a genuine diamond conflict in Python has no "maybe it
     works anyway" middle ground the way an isolated-dependency ecosystem
     sometimes does.
   - In this repository's own stack, `uv`'s resolver generally fails at
     `uv lock`/`uv sync` time with an explicit "cannot resolve" error when
     two dependencies need genuinely incompatible versions of the same
     package, rather than silently picking one and deferring the failure to
     a runtime surprise — a meaningfully better failure mode than the
     classpath-collision model, since the conflict is visible before the
     code ever runs, and a direct consequence of Python's shared-only
     import model making the conflict real rather than something a
     compiled, isolation-capable ecosystem might paper over. Still treat an
     unexpected `uv lock` failure as exactly this scenario, not a transient
     error to retry past.
   - Semantic versioning (major.minor.patch, with a major-version bump
     signaling a breaking change) is what makes automatic resolution
     possible at all: two requirements that only differ in minor/patch
     version can usually be unified onto the newer one safely; two
     requirements on different *major* versions are, by the versioning
     scheme's own contract, potentially incompatible, and forcing a
     resolution between them is exactly where conflicts turn into real
     bugs, silent or not. This is why a library that doesn't follow semantic
     versioning consistently is a real, independent risk factor — its
     version numbers stop being reliable signals for whether an upgrade is
     safe.
   - This is exactly the scenario a lockfile exists to make deterministic
     and visible — resolving a conflict reliably depends on being able to
     see which version won and why, not guessing.

4. **Weigh a library's own dependency footprint in the selection decision**
   - All else equal, prefer a library with fewer, more minimal direct
     dependencies of its own — every dependency it declares becomes part of
     the calling application's dependency graph too, and is itself a future
     source of the exact conflict risk described above.
   - This is a legitimate, concrete evaluation criterion when comparing
     otherwise-similar library candidates, not a minor detail — a library
     that does the same job with a smaller dependency footprint reduces the
     application's total exposure to this class of problem, independent of
     any difference in the libraries' own functionality or API quality.
   - The dependency count also has a direct operational cost, independent of
     conflict risk: every dependency adds to a container image's size,
     which in a Kubernetes deployment affects image pull time, deploy
     speed, and (for anything that scales from zero or restarts frequently)
     cold-start latency. Weigh a library's footprint against this cost too,
     not only against the risk of a version collision.

5. **Recognize dependency isolation techniques as a positive signal, and know what Python's equivalent actually looks like**
   - Some ecosystems let a library isolate its own transitive dependencies
     entirely, so they never collide with a consumer's own versions of the
     same packages — the JVM ecosystem calls this shading (relocating a
     bundled dependency's package namespace at build time). A library that
     does this is signaling real design care about not polluting its
     consumers' dependency graphs; treat it as a positive signal when
     comparing library candidates, the same as a small dependency footprint.
   - Python has no direct equivalent of JVM-style classpath relocation
     (there's no classpath to relocate on), but the same underlying idea
     shows up as **vendoring** — a package bundling a private, internal copy
     of a dependency's source directly inside itself (`pip` famously vendors
     several of its own dependencies this way) instead of declaring it as an
     installable dependency at all. This avoids the conflict entirely
     because there's no separately-installed package name to collide with.
     It's uncommon for anything beyond a handful of foundational tools to
     do this, precisely because of the maintenance overhead described next.
   - Whichever technique is used, it comes with real, ongoing maintenance
     cost for the library's own authors — every isolated/vendored dependency
     needs to be kept up to date manually, including its own security
     patches, since normal dependency-update tooling won't see it. This is
     worth knowing when evaluating why a library might *not* do this, not
     just when praising one that does.

---

## AI decision guidance

When generating dependency-management guidance, keep these principles in
mind:

- **Never treat a transitive dependency as available for direct use** — it's
  an implementation detail of whatever library brought it in, not a
  commitment to the calling application.
- **A direct, explicit dependency is the correct fix**, but it relocates
  version-conflict risk into something visible and resolvable rather than
  eliminating the risk outright.
- **An unexplained runtime error in unrelated-seeming code is a legitimate
  signal to check the resolved dependency tree** for a version conflict, not
  just assume an application-level bug.
- **A library's own dependency footprint is a real selection criterion** —
  fewer, more minimal dependencies reduce the application's total exposure
  to this entire class of problem.

---

## Success criteria

A strong response should ensure that it:

- **never recommends importing a class that's only available as a
  transitive dependency**, and instead recommends declaring a direct
  dependency when the capability is genuinely needed,
- **correctly explains why the transitive-dependency shortcut is risky**
  (an implementation detail the library owner can change without warning),
- **acknowledges that a direct dependency doesn't eliminate conflict risk**,
  and explains how a version conflict actually manifests (a silent
  resolution, then an unexplained runtime error),
- **treats a candidate library's own dependency footprint** as part of a
  fair comparison between alternatives.

---

## Example prompts for the AI

- "Can we just use the JSON library that our HTTP client already pulls in?"
- "We're getting a strange method-not-found error in a library we didn't
  touch — what should we check?"
- "We're choosing between two libraries that do the same thing — does it
  matter that one has way more dependencies than the other?"

---

## Related guidance

Use this tool alongside:

- architecture-code-duplication-tradeoffs — the same transitive-dependency
  coupling cost, in the context of deciding whether to extract a shared
  library in the first place.
- architecture-third-party-defaults-and-concurrency
- architecture-third-party-testability-evaluation
- architecture-third-party-library-selection-checklist — the first-stop checklist tying this skill together with the other third-party-library evaluation areas.
- architecture-semantic-versioning — the SemVer compatibility rules that make automatic conflict resolution possible at all, and why a library that doesn't follow them consistently is an independent risk factor for this skill's diamond-dependency scenario.
