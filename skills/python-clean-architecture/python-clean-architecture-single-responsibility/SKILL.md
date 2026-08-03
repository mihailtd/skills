---
name: python-clean-architecture-single-responsibility
description: Instructs the agent to apply the Single Responsibility Principle at the function/module level, not the class level — a dataclass holding only its core fields, with each group of operations that has its own "reason to change" living in its own module of plain functions. Includes the balance warning against over-fragmenting into too many tiny modules, and why SRP-sized functions are trivial to test.
---

# Python Clean Architecture: Single Responsibility, Functional-Lite

The Single Responsibility Principle (SRP) says a module should have one, and
only one, reason to change. The classic treatment applies this to classes —
splitting a bloated class into several narrower classes. In functional-lite,
the unit of responsibility is a **module of functions operating on a plain
dataclass**, not a class with methods. This skill covers how to apply the
"reason to change" heuristic at that grain, and how to avoid both a
God-object dataclass-with-everything-attached and an over-fragmented mess
of one-function modules.

---

## When to use this skill

Use this skill when you need to:

- decide whether a growing dataclass or module is trying to do too much,
- split a bloated module into focused ones without introducing classes,
- explain the "reason to change" heuristic in terms that fit this repo's
  style,
- judge whether a proposed split is genuinely improving clarity or just
  fragmenting the code needlessly,
- decide how much SRP-driven structure a project actually needs right now
  (see python-clean-architecture-scaling for the broader version of this
  judgment call).

---

## Outcome

Produce a module structure that:

- keeps a domain dataclass reduced to its core, stable fields — the data
  that genuinely belongs to the concept itself, not every operation ever
  performed on it,
- groups functions into separate modules by their distinct "reason to
  change" — operations that would need to change together for the same
  underlying reason live together; operations that change for unrelated
  reasons live apart,
- avoids both extremes: a single module where unrelated concerns are
  tangled together, and a sprawl of one-function modules that makes the
  overall structure harder to navigate than the problem it solves,
- is easy to test in isolation, since each module's functions take plain
  data in and return plain data out, with no dependencies beyond what
  they're actually about.

---

## Instructions for the AI

1. **Keep the domain dataclass reduced to its core fields**
   - When a dataclass accumulates methods or logic for unrelated concerns
     (post creation, timeline generation, profile updates, all attached to
     a `User`), that's the same SRP violation the class-based version
     shows — the fix is different, but the diagnosis is the same.
   - Reduce the dataclass to the fields that genuinely define the concept
     itself:
     ```python
     from dataclasses import dataclass

     @dataclass(frozen=True)
     class User:
         user_id: str
         username: str
         email: str
     ```
   - This is the functional-lite equivalent of the book's refactored,
     stripped-down `User` entity class — same outcome (a focused core
     concept), reached by never attaching behavior to the dataclass in the
     first place rather than by extracting methods out of a class later.

2. **Group functions into modules by distinct reason to change**
   - Translate each of the book's extracted "manager" classes into a
     module of plain functions operating on `User` (and whatever other
     data each concern needs):
     ```python
     # posts.py — reason to change: how posts are created/stored
     def create_post(user: User, content: str) -> Post:
         ...

     # timeline.py — reason to change: how timelines are assembled/ranked
     def get_timeline(user: User) -> list[Post]:
         ...

     # profile.py — reason to change: what profile-update rules apply
     def update_profile(user: User, new_username: str | None = None,
                         new_email: str | None = None) -> User:
         ...  # returns a new User via dataclasses.replace, never mutates
     ```
   - No `PostManager`, `TimelineService`, or `ProfileManager` class exists
     here — the module *is* the grouping mechanism. A module's name should
     describe the responsibility it owns, exactly the way the book's
     class names did, but without needing a class to hold the functions.

3. **Apply the "reason to change" heuristic directly to functions/modules**
   - When deciding whether two functions belong in the same module, ask
     the same question the book asks about classes: if you can think of
     more than one distinct reason either piece of logic would need to
     change, they probably don't belong together.
   - Concretely: "how posts are stored" and "how timelines are ranked" are
     different reasons to change, even though both operate on `User` data
     — keep them in separate modules. "How posts are created" and "how
     posts are validated before creation" are usually the *same* reason to
     change — keep them together.
   - Use this heuristic during review specifically when a module is
     growing large or when a change to one part of a module keeps
     requiring unrelated parts of the same module to be touched or
     re-tested.

4. **Balance SRP against over-fragmentation**
   - Don't mistake SRP for "each function should do only one tiny thing" —
     the principle is about reasons to change, not about minimizing
     function size. Splitting a cohesive piece of logic into several
     one-line functions across several modules, when they all share the
     same reason to change, makes the system harder to navigate, not
     easier.
   - Use judgment: if splitting a module makes the overall structure
     harder to understand rather than easier, that's a signal the split
     wasn't warranted — reverse it.
   - This balance matters more, not less, in functional-lite style, since
     the friction of "creating a new module" is lower than "creating a new
     class" — it's easy to over-apply SRP here specifically because
     splitting is so cheap. Weigh the split against actual navigability,
     not against how easy it was to do.

5. **Let SRP-sized functions demonstrate their own testability**
   - A function scoped to one responsibility, operating on plain
     dataclasses, needs no special setup or mocking to test — this is the
     same benefit the book demonstrates for `PostManager`, achieved here
     without a class to instantiate at all:
     ```python
     def test_create_post():
         user = User("123", "testuser", "test@example.com")
         post = create_post(user, "Hello, world!")
         assert post.user_id == "123"
         assert post.content == "Hello, world!"
         assert post.likes == 0
     ```
   - Treat a function that's hard to test without extensive setup as a
     signal it's carrying more than one responsibility, or that it's
     mixing pure logic with I/O (see
     python-clean-architecture-functional-core-imperative-shell's test-
     friction diagnostic).

6. **Right-size SRP adoption to the project's actual stage**
   - A startup building an MVP might reasonably keep `posts.py` and
     `timeline.py` combined until the combined module actually becomes
     hard to navigate — deferring the split until it's justified by real
     friction, not applying it speculatively on day one.
   - An established system with multiple contributors benefits from
     applying SRP more proactively, since the cost of a tangled module
     compounds faster with more people touching it.
   - See python-clean-architecture-scaling for the fuller treatment of
     this judgment call.

---

## Decision points and guidance

- **Does this dataclass have fields, or does it also carry logic for
  unrelated concerns?** Fields only — extract any attached logic into its
  own module of functions.
- **Do these two functions share a reason to change?** If not, they belong
  in different modules, even if they both operate on the same dataclass.
- **Is a proposed split improving navigability, or just adding module
  count?** Only proceed with the split if it genuinely clarifies the
  structure.
- **Is this function hard to test without heavy setup?** Treat that as a
  signal of a responsibility (or a pure/impure) boundary problem, not just
  a testing inconvenience.
- **Does the project's current stage justify this split yet?** Apply SRP
  in proportion to actual friction, not speculative future need.

---

## Quality criteria

A strong SRP-applied module structure should ensure that:

- **domain dataclasses hold only their core fields**, with no attached
  behavior for unrelated concerns,
- **each module groups functions by a genuinely distinct reason to
  change**, not by superficial similarity,
- **splits are justified by real navigability gains**, not applied
  reflexively because splitting is cheap,
- **every function is testable with plain data in, plain data out**, with
  no setup beyond constructing the inputs,
- **the granularity matches the project's actual stage**, deferred where
  premature, applied proactively where multiple contributors are already
  feeling the friction of a tangled module.

---

## Example prompts

- "This module is doing too much — help me split it by reason to change,
  without introducing classes."
- "Is this split actually making the code easier to navigate, or did we
  just create five tiny modules for no reason?"
- "This function is hard to test — is that an SRP problem or a pure/impure
  boundary problem?"

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-functional-core-imperative-shell
- python-clean-architecture-dependency-rule
- python-clean-architecture-scaling
- python-clean-architecture-domain-modeling
- python-clean-architecture-entity-invariants
