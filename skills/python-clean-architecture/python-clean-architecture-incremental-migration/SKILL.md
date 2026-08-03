---
name: python-clean-architecture-incremental-migration
description: Instructs the agent to safely cut a running system over from legacy code to a functional-lite Clean Architecture implementation — regression tests captured before any refactor begins, then Strangler Fig / feature-flag parallel implementation / Shadow Mode to let old and new coexist, each with an explicit parallel-operation strategy, verification approach, cutover criteria, and rollback procedure. This is the execution-safety companion to python-clean-architecture-legacy-assessment, which covers deciding what to migrate and in what order.
---

# Python Clean Architecture: Migrating Incrementally Without Breaking Production

Once a transformation is planned (see python-clean-architecture-legacy-assessment),
executing it against a live system is a distinct problem: the system must
keep working throughout, for every stage. This skill covers the concrete
patterns for letting old and new implementations coexist safely — the
Strangler Fig pattern, feature-flag-gated parallel implementations, Shadow
Mode, and the adapter pattern for bridging — plus the four things every
in-flight component needs a plan for: how old and new coexist, how you
verify they're equivalent, what triggers cutover, and how you roll back.

---

## When to use this skill

Use this skill when you need to:

- cut a specific piece of legacy functionality over to a new Clean
  Architecture implementation without a "big bang" release,
- decide between the Strangler Fig pattern, feature-flag parallel
  implementation, and Shadow Mode for a specific migration,
- establish regression test coverage before refactoring legacy code,
- define cutover and rollback criteria for an in-flight component,
- bridge a new domain model to an existing route handler or entry point
  during the transition period.

---

## Outcome

Produce a migration that:

- never attempts a "big bang" replacement — the legacy and new
  implementations coexist for a controlled period, with an explicit exit
  condition,
- is protected by regression tests captured *before* any refactoring
  begins, verifying the existing behavior the new implementation must
  match,
- has, for every component being migrated, an explicit answer to four
  questions: how do old and new coexist, how do we verify they're
  equivalent, what triggers cutover, and how do we roll back,
- uses the pattern (Strangler Fig, feature flags, Shadow Mode, adapter)
  that actually fits the component's risk profile, not one pattern applied
  uniformly everywhere.

---

## Instructions for the AI

1. **Capture regression tests before touching legacy code**
   - Before refactoring anything, establish a test suite that documents
     the *existing* system's actual behavior — these tests exist to catch
     unintended behavior changes during the transformation, not to verify
     the new architecture is "correct" in the abstract.
   - Start with end-to-end tests covering complete workflows through the
     legacy system (see python-clean-architecture-testing-strategy for
     where end-to-end tests fit in the broader test-distribution
     strategy), then add more granular tests as clean boundaries emerge —
     the reverse order from greenfield development, where unit tests come
     first.
   - Treat these regression tests as a genuine safety net: they're what
     lets the team refactor with confidence, and what gives stakeholders
     evidence the transformation isn't silently breaking things.

2. **Choose an in-flight coexistence pattern deliberately**
   - **Adapter pattern:** wrap legacy components behind the new domain
     interfaces so old and new can interact — this is the same
     `Callable`-type adaptation already established (see
     python-clean-architecture-drivers), applied to bridge *existing* code
     rather than a fresh third-party service.
   - **Feature-flag parallel implementation:** run the new implementation
     alongside the legacy one, with a flag controlling which handles a
     given request — provides an immediate, low-risk rollback (flip the
     flag) if issues surface.
   - **Strangler Fig:** incrementally replace pieces of the legacy
     application while preserving the same external interface, until the
     old implementation can be safely deleted — good for larger
     components that can't be swapped in one step.
   - **Shadow Mode:** duplicate real requests to the new implementation
     via a proxy, let it process its own copy, and compare outputs against
     the legacy result *without* affecting what the user actually
     receives — validates real-world behavioral equivalence before any
     user-facing cutover risk at all.
   - Match the pattern to the component's risk: Shadow Mode for
     high-stakes components where a live bug would be costly; feature-flag
     parallel implementation for components where a fast, simple rollback
     is enough; Strangler Fig for large components that must be replaced
     piece by piece rather than atomically.

3. **Answer four questions explicitly for every component in flight**
   - **Parallel operation strategy:** exactly how do the old and new
     implementations coexist while both exist? (Which pattern from step 2,
     and how is routing between them decided?)
   - **Verification approach:** how do we confirm the new implementation
     is actually equivalent — regression tests, Shadow Mode output
     comparison, or both?
   - **Cutover criteria:** what specific, concrete condition triggers
     switching real traffic to the new implementation (a time-boxed Shadow
     Mode period with zero discrepancies, a target regression-test pass
     rate, explicit stakeholder sign-off)?
   - **Rollback procedure:** what's the actual mechanism to revert if
     something goes wrong after cutover (flip the feature flag back,
     redeploy the prior version, restore from a specific point)? This must
     be a real, tested mechanism, not an assumption that reverting will be
     straightforward.
   - Write these down per component before starting its migration — don't
     let "we'll figure out rollback if we need it" stand in for an actual
     answer.

4. **Bridge a legacy entry point to a new controller function via a feature flag**
   - Modify the existing entry point to delegate to the new,
     already-reformulated controller function (see
     python-clean-architecture-controllers) behind a flag, rather than
     replacing the entry point outright:
     ```python
     @app.route('/orders', methods=['POST'])
     def create_order():
         data = request.get_json()
         if not data or 'customer_id' not in data or 'items' not in data:
             return jsonify({'error': 'Missing required fields'}), 400

         if app.config.get("USE_CLEAN_ARCHITECTURE", False):
             result = handle_create_order(create_order_use_case, present_order, data)
             match result:
                 case Success(view_model):
                     return jsonify(view_model), 201
                 case Failure(error):
                     return jsonify({"error": error.message}), 400
         else:
             # ... original implementation remains, untouched, until cutover ...
     ```
   - Note that `handle_create_order` and `create_order_use_case` here are
     already the plain functions established throughout this package
     (python-clean-architecture-use-cases,
     python-clean-architecture-controllers) — the migration-specific part
     is entirely the feature-flag branch in the legacy entry point, not a
     new production-code pattern. Once cutover criteria are met, delete
     the `else` branch and, eventually, the flag itself.

5. **Track migration progress the same way the roadmap tracks it**
   - Tie each in-flight component's cutover back to the baseline metrics
     established during planning (see
     python-clean-architecture-legacy-assessment) — a successful cutover
     should show up as sustained or improved values on those metrics, not
     just "the flag is flipped."
   - Remove dead legacy code and flags promptly once cutover criteria are
     met and the rollback window has passed — a flag left in place
     indefinitely after cutover is itself a small architectural violation
     (permanent conditional complexity for a decision that's already been
     made).

---

## Decision points and guidance

- **Has this component's regression test coverage been captured before
  any refactoring started?** If not, do that first — it's the actual
  safety net for everything that follows.
- **Which coexistence pattern fits this component's risk?** High-stakes,
  hard-to-reverse consequences favor Shadow Mode; moderate risk with a
  need for fast rollback favors feature-flag parallel implementation;
  large components favor Strangler Fig's piece-by-piece replacement.
- **Are all four questions (coexistence, verification, cutover, rollback)
  answered concretely for this component?** If any is vague, treat that as
  unfinished planning, not an acceptable gap.
- **Has a feature flag outlived its purpose?** If cutover happened and the
  rollback window passed, remove the flag and the legacy branch — don't
  let it linger.

---

## Quality criteria

A strong incremental migration should ensure that:

- **regression tests exist before refactoring starts**, documenting the
  legacy behavior the new implementation must match,
- **the coexistence pattern matches the component's actual risk profile**,
  not applied uniformly by default,
- **every in-flight component has explicit, concrete answers** to
  coexistence, verification, cutover, and rollback,
- **legacy branches and feature flags are removed promptly** once cutover
  is confirmed successful, not left as permanent complexity,
- **cutover success is measured against the baseline metrics** established
  during planning, not just "the flag is on."

---

## Example prompts

- "Help me set up a feature-flag-gated parallel implementation for this
  order creation endpoint before cutting over."
- "Is this component risky enough to warrant Shadow Mode instead of just a
  feature flag?"
- "We haven't defined rollback criteria for this migration — help me work
  that out before we start."
- "This feature flag has been in place for months after cutover — help me
  clean it up."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-legacy-assessment
- python-clean-architecture-testing-strategy
- python-clean-architecture-controllers
- python-clean-architecture-drivers
- python-clean-architecture-composition-root
