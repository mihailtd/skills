---
name: architecture-trend-adoption-discipline

description: Instructs the AI assistant to verify a team actually has the problem a trendy framework or paradigm claims to solve before adopting it — a solution's hidden complexity scales with how many problems it solves, so importing a powerful framework for one narrow need still pays for everything else it does; that framework adoption cascades (one framework choice often forces compatible siblings, e.g. a DI framework pulling in its matching web framework), silently widening the blast radius past the original decision; and that an end-to-end paradigm shift (async/streaming, reactive) can't be partially adopted — once a function signature commits to it, the commitment leaks to every caller, so "just this one component" is rarely how it stays.
---

# Trend Adoption Discipline Instructions

When supporting a decision to adopt a new framework, library, or programming
paradigm — especially one that's currently popular or being pushed by
"everyone else uses it" pressure — use this tool to check that the decision
is driven by a measured, actual problem rather than by trendiness itself.

---

## Purpose

This tool helps the AI assistant by:

- requiring that the specific problem a trendy solution claims to solve is
  first confirmed to actually exist in the system under discussion, before
  any framework or paradigm adoption is recommended,
- making the case that a solution's internal complexity is proportional to
  how many problems it's built to solve — adopting a broad, powerful
  framework for one narrow need still imports everything else it does as
  permanent overhead, not a free capability sitting unused,
- flagging that framework adoption cascades — choosing one framework
  frequently forces adopting its compatible siblings to get the promised
  behavior at all, which silently expands the actual blast radius well past
  the original, narrower-sounding decision,
- distinguishing a full end-to-end paradigm migration from a "just this one
  component" adoption — a paradigm that changes what a function's signature
  promises (returns a stream instead of a value, requires being awaited,
  requires a wrapping runtime) leaks that requirement to every caller, so a
  partial adoption is usually a false economy, not a smaller, safer step.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- names the concrete, currently-experienced problem before recommending a
  trendy framework or paradigm as the fix, and treats "it's popular" or
  "it's the modern way to do this" as irrelevant on their own,
- weighs a candidate solution's full complexity — not just the complexity of
  the one feature being sought — against the complexity of the problem it's
  meant to solve, and says so explicitly when the two are mismatched,
- traces what else a framework choice will require adopting to actually work
  as advertised, and treats that full chain as the real cost of the
  decision, not just the first framework named,
- decides explicitly, up front, whether a paradigm shift is a full,
  end-to-end migration or is staying scoped to one component — and if
  scoped, verifies the paradigm's constraints can actually be contained at
  that boundary rather than assuming they will be,
- considers a smaller, narrower, DIY, or already-available primitive (a
  plain factory function, the language's built-in concurrency primitives)
  before reaching for a purpose-built framework or paradigm, and only moves
  past it when a genuine, named limitation of the simpler option shows up.

---

## Instructions for the AI

1. **Confirm the problem exists before recommending the trendy solution**
   - Before recommending a framework, library, or paradigm that's currently
     popular, ask what specific, currently-experienced problem it would
     solve for this system — not what problem it solves in general, or what
     problem it solved for whoever is advocating for it.
   - If the honest answer is "we don't have that problem yet, but we might
     eventually" or "it's what everyone uses now," treat that as a reason to
     hold off, not a reason to adopt early. Popularity and trendiness are not
     evidence that a solution fits this system's actual, current pain.
   - When the problem does genuinely exist, name it concretely and use that
     as the basis for evaluating candidate solutions — including whether a
     much smaller fix addresses it just as well (point 5).

2. **Weigh a solution's total complexity against the problem it actually solves**
   - A framework or paradigm's internal complexity roughly scales with how
     many problems it's designed to solve, not with how many of those
     problems a given adopter actually has. A full dependency-injection
     container with configurable bean scopes, lifecycle hooks, and proxy
     based interception solves several distinct problems at once — but a
     team adopting it only to get "one instance per request" pays for the
     rest of that machinery regardless of whether they ever use it.
   - Make this mismatch explicit when it exists: "this framework solves five
     problems, we have one of them" is a legitimate reason to look for a
     narrower fix, even when the framework is genuinely well-built and
     well-regarded.

3. **Trace what else the choice will actually require adopting**
   - A framework choice is rarely fully self-contained — check what else it
     requires to deliver the specific behavior being sought. A dependency
     injection framework's request-scoped component lifecycle, for example,
     is often only fully realized when paired with that same framework's own
     web framework — adopting the DI container alone doesn't get you there,
     and the "second" adoption (the web framework) is usually not optional
     once the first commitment is made.
   - Treat this full chain — not just the first named framework — as the
     real cost of the decision, and evaluate it with the framework-level
     scrutiny **architecture-third-party-library-selection-checklist**
     already recommends for any framework choice (substantially more
     up-front investigation than a library, since frameworks are
     structurally harder to walk back).

4. **Decide explicitly: full paradigm migration, or genuinely contained at one boundary**
   - Some paradigm shifts change what a function's own signature promises —
     returning a lazily-evaluated stream instead of a concrete value,
     requiring the caller to be inside an async runtime, requiring the
     caller to understand backpressure or a scheduler model. Once a
     signature makes that promise, every caller has to honor it; there's
     usually no safe way to "unwrap" back to a plain synchronous value
     without either blocking indefinitely (defeating the paradigm's own
     point) or duplicating the logic in two forms.
   - Decide up front, explicitly, whether the goal is an end-to-end
     migration (the whole call chain, producer to final consumer, adopts
     the paradigm) or a change genuinely contained within one component's
     boundary. If it's meant to stay contained, verify that the paradigm's
     constraints can actually be absorbed at that boundary — a lightweight
     async/concurrency primitive that can be trivially awaited or collected
     into a plain value by its caller (Python's `asyncio.gather` /
     `TaskGroup`, a plain `Future`) usually can; a full reactive-streams
     abstraction usually can't, because "give me the streaming API, but let
     everyone else keep using it synchronously" is close to a contradiction
     in terms.
   - When only a narrow slice of processing needs to be non-blocking or
     parallelized, prefer the primitive that stays easy to collapse back to
     a plain value for callers that don't need the paradigm — see
     **python-asyncio-concurrent-web-requests** for the concrete Python
     shape of this (structured concurrency via `asyncio.gather`/`TaskGroup`
     rather than an async-generator/streaming redesign of the whole call
     path).

5. **Consider the DIY or already-available option before the framework**
   - Many problems reached for via a framework have a much smaller, local
     fix: a plain factory function that constructs a fresh instance per
     call for "one instance per request," the standard library's own
     concurrency primitives for "run these concurrently," a dataclass and a
     handful of functions for "manage this small piece of state." These
     aren't inferior stand-ins to eventually replace — they're often the
     entire correct solution, staying that way as long as the problem stays
     small.
   - Move to a heavier, purpose-built framework only when a specific,
     named limitation of the simpler option actually shows up in practice —
     not preemptively, on the assumption that the limitation will
     eventually matter. This is the same "don't design for hypothetical
     future requirements" discipline **architecture-simplicity** and
     **architecture-flexibility-complexity-tradeoffs** apply elsewhere,
     applied here specifically to framework and paradigm adoption
     decisions.

---

## AI decision guidance

When generating trend-adoption guidance, keep these principles in mind:

- **A trendy solution needs a named, currently-real problem to justify
  adopting it** — popularity is not evidence of fit for this system.
- **A solution's complexity is priced for all the problems it solves, not
  just the one you have** — a mismatch between the two is a legitimate
  reason to look elsewhere.
- **Framework adoption cascades** — trace the full chain of what else a
  choice will require, and price that whole chain, not just the first named
  framework.
- **A paradigm that changes a function's own contract can't usually be
  adopted halfway** — decide explicitly whether the change is end-to-end or
  genuinely containable at one boundary, and verify the containment claim
  rather than assuming it.
- **Prefer the smaller, already-available option until a specific,
  named limitation of it actually shows up** — don't reach for a framework
  preemptively.

---

## Success criteria

A strong response should ensure that it:

- **names the specific, currently-real problem** a trendy adoption is meant
  to solve, and treats trendiness alone as insufficient justification,
- **compares the candidate's total complexity to the actual problem size**,
  flagging a mismatch explicitly rather than adopting a powerful solution
  for a narrow need,
- **traces the full chain of what the choice will require adopting**, not
  just the first-named framework,
- **makes an explicit end-to-end-vs-contained decision** for any paradigm
  shift that changes a function's own contract, and checks that a
  "contained" claim is actually true,
- **starts from the smaller, already-available option** and only escalates
  to a heavier framework once a concrete limitation of the simpler approach
  has actually appeared.

---

## Example prompts for the AI

- "Should we bring in a dependency injection framework, or is our current
  manual wiring fine?"
- "The team wants to rewrite our data pipeline using a reactive/streaming
  library — is that worth it?"
- "We only need to parallelize one slow part of this request — do we need
  to redesign it around async generators?"
- "Everyone on the team keeps mentioning [trendy framework] — should we be
  using it?"

---

## Related guidance

Use this tool alongside:

- architecture-third-party-library-selection-checklist — the deeper,
  framework-specific evaluation (health, licensing, security) this skill's
  point 3 leans on once a framework is genuinely being considered; this
  skill is the earlier gate that decides whether that evaluation is even
  warranted.
- architecture-simplicity / architecture-flexibility-complexity-tradeoffs —
  the general "don't design for hypothetical future requirements"
  discipline this skill applies specifically to framework and paradigm
  adoption.
- python-fastapi-dependency-injection — the built-in DI mechanism
  (`Depends`) FastAPI already provides; this repo's stack doesn't face the
  "adopt an external DI framework" decision this skill's DI example
  illustrates, since FastAPI's own is already the answer for that specific
  case.
- python-asyncio-concurrent-web-requests — the concrete, containable Python
  primitive (`asyncio.gather`/`TaskGroup`) this skill's point 4 recommends
  reaching for before a full reactive/streaming redesign.
- python-functional-programming-recursion-tco — a related but distinct
  caution about importing a language paradigm (unbounded recursion, common
  in functional languages) into a language whose runtime doesn't support it
  the same way; that skill covers the mechanical fix, this one covers the
  earlier "should we" decision for adopting a paradigm at all.
