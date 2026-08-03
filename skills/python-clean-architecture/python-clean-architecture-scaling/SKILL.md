---
name: python-clean-architecture-scaling
description: Instructs the agent to treat Clean Architecture as a spectrum of independently-adoptable patterns, not an all-or-nothing structure — right-sizing adoption to a Python project's actual size and complexity, starting small projects with a simple but modular layout (separate business logic from presentation, inject dependencies, define clear interfaces) and evolving toward the full layered structure only as growth actually justifies it. Prevents both under-engineering (tangled small scripts) and over-engineering (full four-ring ceremony for a throwaway script).
---

# Python Clean Architecture: Scaling the Pattern to the Project

Clean Architecture is a spectrum of independently valuable patterns, not a
binary choice between "no architecture" and "the full four-ring structure."
This skill covers how to judge which patterns actually earn their cost for
a given project's size and complexity, how to start small without painting
yourself into a corner, and how to evolve toward the full structure only as
real growth justifies it — avoiding both a tangled small script and a
needlessly ceremonial one.

---

## When to use this skill

Use this skill when you need to:

- decide how much Clean Architecture structure a new project or script
  actually needs before writing any code,
- review a small project that's either too tangled to extend safely, or
  suspiciously over-structured for what it does,
- plan how an existing small project should evolve its structure as it
  grows in scope,
- push back on (or defend) a proposal to introduce full layering into a
  project that doesn't yet need it,
- explain to a team why a data-processing script and a production API
  should adopt different subsets of Clean Architecture patterns.

---

## Outcome

Produce a project structure that:

- applies only the Clean Architecture patterns that provide clear value
  for the project's actual size, complexity, and expected lifespan — never
  the full four-ring ceremony by default,
- keeps even a small project's core business logic separated from
  presentation/I/O concerns, using plain functions and parameter-passing,
  from the very first version — this costs almost nothing and prevents the
  tangle that makes later growth expensive,
- has a clear, low-friction path to evolve toward the full layered
  structure (see python-clean-architecture-dependency-rule) if and when
  the project's scope genuinely grows to justify it,
- avoids over-engineering: a throwaway script or a small internal tool
  should not carry a full `entities/`/`use_cases/`/`interfaces/`/
  `frameworks/` directory tree it doesn't need.

---

## Instructions for the AI

1. **Treat Clean Architecture as a spectrum, not a binary**
   - Reject the framing that a project either "does" or "doesn't do" Clean
     Architecture. Instead, evaluate each pattern independently — pure
     domain functions, dependency inversion via `Callable` types, a full
     layer-package directory structure, interface-adapter translation
     layers — and adopt only the ones that provide clear value for this
     specific project.
   - Concrete examples of partial adoption that are legitimate, not
     "doing it wrong": a small API might adopt clean, pure use-case
     functions without a full presenter-layer abstraction; a
     data-processing script might adopt immutable domain dataclasses and
     pure transformation functions while skipping interface adapters
     entirely, since there's no varying "interface" to adapt for.
   - When advising on structure, name specifically which patterns are
     being adopted and which are being deliberately skipped, and why —
     this keeps the decision legible and revisitable, rather than an
     implicit, undiscussed simplification.

2. **Give small projects a minimal-but-modular starting structure**
   - For a small project or quick prototype, a full layer-package
     directory tree is usually not worth its overhead on day one. Instead,
     recommend a minimal structure that still separates concerns:
     ```
     project/
       models.py     # immutable dataclasses — domain concepts
       logic.py       # pure functions — business rules over the models
       io.py           # plain functions — the only place doing I/O
       main.py        # entry point — wires io.py output into logic.py,
                        #   passes logic.py output to io.py
     ```
   - Even at this minimal scale, keep applying the core practices from
     python-clean-architecture-functional-core-imperative-shell: business
     logic (`logic.py`) stays pure and separate from I/O (`io.py`); I/O
     functions are passed into wherever they're needed as plain
     parameters rather than imported and called directly deep inside
     `logic.py`; boundaries between modules are defined by function
     signatures using plain types, not ad hoc dict-passing.
   - This minimal structure costs almost nothing to set up correctly the
     first time, and is what makes the next step (evolving toward a fuller
     structure) a mechanical refactor instead of a rewrite.

3. **Evolve toward the full structure only as growth justifies it**
   - Recommend growing the structure in this order, only taking each step
     when the project has actually outgrown the previous one (more domain
     concepts than fit legibly in one `models.py`, more use cases than fit
     in one `logic.py`, more than one delivery mechanism, more than one
     infrastructure integration):
     1. Split `models.py` and `logic.py` into `entities/` and `use_cases/`
        packages once there are enough distinct concepts/use cases that a
        single file no longer reads clearly.
     2. Introduce explicit `Callable`-typed abstractions (see
        python-clean-architecture-dependency-inversion) in an
        `interfaces/` package once more than one concrete implementation
        of a dependency needs to exist (e.g., a real database and an
        in-memory fake for tests, or two delivery mechanisms).
     3. Organize `tests/` to mirror the layer structure once the test
        suite itself has grown large enough that flat organization makes
        it hard to tell which layer a given test covers.
   - Frame each step as justified by a concrete, present need — not
     "because bigger projects have this structure" — so the team can
     recognize when they've actually reached the trigger for the next
     step, rather than restructuring speculatively ahead of need (see
     architecture-simplicity's caution against future-proofing, which
     applies equally at this smaller scale).

4. **Keep the core discipline even when skipping structural ceremony**
   - The one thing not to skip at any scale: keeping business-decision
     code pure and separate from I/O code. This is cheap even in a
     one-file script (a `logic.py` next to an `io.py`) and expensive to
     retrofit later if skipped — unlike the directory/package structure,
     which is comparatively cheap to introduce later once the project
     actually needs it.
   - When reviewing a small project, distinguish between "this correctly
     skipped full layering because it doesn't need it yet" and "this
     tangled I/O and business logic together, which will be expensive to
     untangle regardless of directory structure" — only the first is an
     acceptable simplification.

---

## Decision points and guidance

- **Does this project need the full four-ring structure right now, or just
  the core/shell discipline?** Default to the minimal structure that still
  keeps logic pure and I/O separate; only add layer packages when a
  concrete trigger (too many concepts, multiple implementations, a
  growing test suite) justifies it.
- **Is a proposed simplification skipping ceremony, or skipping the core
  discipline?** Skipping `entities/`/`use_cases/` packages for a one-file
  script is fine; mixing business decisions into I/O functions is not,
  regardless of project size.
- **Has the project actually hit a growth trigger, or is restructuring
  being done speculatively?** Only split into fuller structure when a
  concrete, present need (not an anticipated future one) justifies it.
- **Are different parts of the same system reasonably adopting different
  levels of structure?** That's expected and fine — a stable core service
  might warrant the full structure while an adjacent small script doesn't;
  judge each independently.

---

## Quality criteria

A strong scaled Clean Architecture adoption should ensure that:

- **pattern adoption is explicit and justified**, not an implicit
  all-or-nothing decision,
- **small projects stay simple in structure but disciplined in
  separation** — pure logic separated from I/O even in a one-file script,
- **structural growth follows concrete triggers**, not anticipated future
  scale,
- **the core/shell discipline is never skipped**, even when directory
  structure and interface-adapter ceremony are,
- **different parts of a system can carry different levels of structure**
  without that being treated as inconsistency.

---

## Example prompts

- "This is a small internal script — how much Clean Architecture structure
  does it actually need?"
- "Our prototype is growing — help me figure out if it's time to split
  into entities/use_cases packages yet."
- "Someone wants to add a full four-layer directory structure to this
  50-line script — help me push back with the right reasoning."
- "This one-file script mixes business logic and I/O — is that an
  acceptable simplification, or a real problem?"

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-functional-core-imperative-shell
- python-clean-architecture-dependency-rule
- python-clean-architecture-dependency-inversion
- architecture-simplicity
- architecture-incremental-delivery
