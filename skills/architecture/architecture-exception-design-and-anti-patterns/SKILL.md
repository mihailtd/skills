---
name: architecture-exception-design-and-anti-patterns

description: Instructs the AI assistant on exception-handling design in Python and JavaScript/TypeScript — since neither language has Java-style checked exceptions, replacing that compiler-enforced contract with a documented/typed expected-failure return; catching at the granularity the caller's handling actually needs; wrapping third-party exceptions into domain-specific ones; and correcting the "don't print the stack trace" rule for a Kubernetes+Datadog context, where stdout is the intended log pipeline and the real fix is routing exceptions through the structured logger instead of print()/console.log().
---

# Exception Design and Anti-Patterns Instructions

When supporting exception-handling design or review in Python or
JavaScript/TypeScript, use this tool to translate exception-hierarchy reasoning
that assumes Java-style checked exceptions into idioms that actually work in
languages with no compiler-enforced exception contract, to calibrate catch
granularity correctly per language, and to correct a common but
context-dependent piece of advice — "never print a stack trace" — for this
repository's actual deployment target (Kubernetes, stdout shipped to Datadog).

---

## Purpose

This tool helps the AI assistant by:

- recognizing that neither Python nor JavaScript/TypeScript has Java-style
  checked exceptions enforced by the compiler, and translating the underlying
  goal — callers should know what a function can fail with, and expected
  failures should be handleable without reading the implementation — into a
  typed Result/discriminated-union return for failures callers must branch on,
  reserving raised/thrown exceptions for genuinely exceptional cases,
- calibrating catch granularity using each language's real syntax and
  gotchas — Python's `except (A, B) as e:` tuple form and the
  bare-`except:`-also-catches-`SystemExit`-and-`KeyboardInterrupt` trap; JS's
  single `catch` clause, which requires `instanceof` branching to
  differentiate,
- wrapping a third-party library's exception into a domain-specific one at a
  public boundary, preserving the original as the cause rather than either
  leaking the third-party type or discarding the underlying information,
- naming and correcting the concrete anti-patterns: swallowing an exception,
  using exceptions for expected control flow, logging the same failure at
  every layer it passes through instead of once, and — the one needing the
  sharpest correction for this repository specifically — treating "avoid
  stdout" as the point of "don't print a stack trace," when the actual
  problem is bypassing the structured logger, not the destination.

---

## Expected outcome

As the AI, your response should help produce exception-handling code that:

- represents "the caller is expected to handle this" failures as an explicit
  typed return (Result/discriminated union), and reserves raised/thrown
  exceptions for failures the immediate caller can't sensibly act on,
- catches only as broadly as the caller's differentiated handling requires,
  never a bare `except:`/empty `catch {}` used to make an error disappear,
- never lets a third-party exception type leak through a module's or
  service's own public interface — it's wrapped in a domain-specific
  exception with the original preserved as the cause,
- routes every caught exception meant to be observed through this project's
  structured logger (see **python-configuration-observability**), never
  `print()`/`console.log()`/an unguarded stack-trace dump — understood as a
  correction about *format*, not about *destination*,
- cleans up resources (files, connections, HTTP clients) unconditionally, on
  both the success and failure path, via a context manager (`with`) in Python
  or `try/finally` in JS — never a `close()` call placed after risky code
  inside the same `try` block with nothing guaranteeing it runs,
- composes multiple fallible steps via chaining rather than unwrapping and
  re-checking a Result after every call, and logs a given failure exactly
  once, at the layer that actually handles it.

---

## Instructions for the AI

1. **Replace "checked exceptions" with a documented, typed contract**
   - Neither Python nor JS/TS enforces at compile time what a function can
     raise/throw — don't try to fake that enforcement with comments alone.
     For failures a caller is expected to actually handle (not just log and
     let crash), prefer returning an explicit result type instead of raising:
     - **Python:** the `Result[T]` tagged-union pattern — `Success[T] |
       Failure`, returned rather than raised — see
       **python-domain-error-handling** for the full pattern and why it must
       be a tagged union, not a single dataclass with optional fields.
     - **JS/TS:** the equivalent discriminated-union return, `{ ok: true,
       value: T } | { ok: false, error: E }` (or a small `Result<T, E>` helper
       if the codebase already has one) — same shape, same reasoning: a type
       checker can force the caller to handle both branches; a thrown
       exception cannot.
   - Reserve actually raising/throwing for failures the immediate caller
     can't sensibly handle — a bug, a violated precondition, an unrecoverable
     environment failure — and document those in a docstring (Python) or
     JSDoc `@throws` (TypeScript) for human readers, since nothing enforces
     it mechanically in either language.
   - The underlying reason a return-typed failure beats an exception isn't
     stylistic: an exception is a side effect hidden from a function's
     signature (unless the language enforces declaring it, which neither of
     these does), while a Result/discriminated-union return makes failure
     part of the value the caller receives — there's no way to "forget" to
     consider it the way there is with an undeclared `raise`/`throw`.
   - Pick one default per boundary/layer — Result-returning or
     exception-raising — and stay consistent within it. A layer that mixes
     both (some functions return `Result`, others throw unchecked) forces
     every caller to handle two different failure-signaling mechanisms at
     once, which is strictly worse than either approach used consistently.
     The usual shape that works well: catch exceptions from impure/third-party
     code at the boundary of a layer, translate them into a `Result`/`Err`,
     and keep everything inside that layer exception-free — see
     **python-domain-error-handling** section 4 for exactly this pattern
     applied at a Clean Architecture use-case boundary.

2. **Chain Result-returning steps instead of re-checking after each one**
   - The value of modeling failure in the return type disappears if every
     caller immediately unwraps it and branches with `if`/`else` at each
     step — that's exceptions with extra ceremony. Compose fallible steps so
     a failure short-circuits the chain instead of requiring a manual check
     after every call:
     - **Python:** pattern-match once, at the point a value is actually
       needed, rather than after every intermediate step:
       ```python
       match fetch_user(user_id):
           case Success(user):
               ...
           case Failure(error):
               logger.error("fetch_user failed: %s", error)
               return Failure(error)
       ```
       If a pipeline genuinely has several sequential fallible steps, a small
       `map`/`and_then` helper on the project's `Result` type lets them
       compose without unwrapping early — the same shape as `map`/`mapTry`
       chaining on a Try/Result monad, just without requiring a full
       monadic library.
     - **JS/TS:** a `Result<T, E>` type with `.map()`/`.andThen()` (as
       libraries like `neverthrow` provide, or a small hand-rolled
       equivalent) lets a pipeline of fallible steps read as one chain,
       short-circuiting on the first failure instead of a manual `if
       (!result.ok) return result;` after every call:
       ```typescript
       const result = parseInput(raw).andThen(validate).andThen(save);
       if (!result.ok) {
         logger.error({ err: result.error }, "pipeline failed");
       }
       ```
   - Don't introduce chaining machinery for a single fallible step — a plain
     `match`/`if` is clearer there. Reach for composition once a sequence of
     steps would otherwise mean repeating the same unwrap-and-check block
     several times in a row.

3. **Catch at the granularity the caller's handling actually needs**
   - **Python:** use one specific `except SpecificError as e:` per error that
     needs distinct handling. Use the tuple form, `except (ErrorA, ErrorB) as
     e:`, only when both need identical handling — Python's equivalent of
     Java's multi-catch. Never use a bare `except:` — it also catches
     `SystemExit` and `KeyboardInterrupt`, which are not ordinary errors and
     must not be silently absorbed; write `except Exception as e:` if the
     intent is genuinely "catch anything unexpected."
   - **JavaScript/TypeScript** has exactly one `catch` clause per `try` —
     there's no multi-catch syntax. Differentiate by branching inside the
     single `catch` with `instanceof`:
     ```javascript
     try {
       await doSomething();
     } catch (e) {
       if (e instanceof NetworkError) {
         logger.warn({ err: e }, "network problem, will retry");
       } else if (e instanceof ValidationError) {
         logger.error({ err: e }, "invalid input");
       } else {
         throw e; // not one we know how to handle — let it propagate
       }
     }
     ```
   - In both languages, don't broaden a catch just to shrink the block count —
     a catch-all silently absorbs errors the caller never intended to handle,
     including ones that should propagate to a higher layer.

4. **Wrap third-party exceptions instead of leaking their type**
   - When a public function's failure mode is really "the underlying library
     failed," don't let that library's specific exception type leak through
     your own interface — a version bump or a library swap then becomes a
     breaking change for every caller. Wrap it in a domain-specific
     exception, preserving the original as the cause:
     - **Python:** `raise PersonCatalogError("...") from e` — `from e` sets
       `__cause__`, so the original traceback stays visible in logs and
       tracebacks even though it's no longer part of the public type
       contract. This is the default for a *meaningful* underlying failure
       worth keeping visible. Reserve `from None` (see
       **python-domain-error-handling**'s exception-concealment guidance) for
       the narrower case where the low-level cause is pure implementation
       noise (e.g. an internal `KeyError`) that would only confuse a
       domain-layer consumer — that's a deliberate exception to this
       skill's default, not a contradiction of it.
     - **JS/TS:** `throw new PersonCatalogError("...", { cause: e })` — the
       `cause` option (ES2022+) does the same job. On a runtime without it,
       attach the original as a plain property (`err.cause = e`) rather than
       dropping it.
   - This is worth the small amount of extra code specifically at a
     public/library boundary — internal, private functions calling other
     internal functions in the same codebase don't need this insulation,
     since a caller there can already see the real implementation.

5. **Don't use exceptions for expected control flow**
   - If a "failure" is a normal, expected outcome the caller will branch on
     nearly every call ("value not found," "validation failed"), that's a
     signal to return a Result/discriminated union (point 1) rather than
     raise/throw — using exceptions here makes the caller's branching logic
     harder to read, and in both Python and JS is meaningfully slower than a
     plain conditional.
   - Reserve raising/throwing for outcomes genuinely exceptional relative to
     the normal operation of the calling code — not every "no" a function can
     answer with.

6. **Never swallow an exception — and log through the structured logger, not `print()`/`console.log()`**
   - Swallowing (`except Exception: pass` / `catch (e) {}`) silently discards
     evidence of a real failure — never do this. If a caught exception
     genuinely needs no action, that's still worth a one-line log explaining
     *why* it's safe to ignore, not silence.
   - **On "don't print the stack trace":** this repository runs in
     Kubernetes with stdout captured and shipped to Datadog automatically
     (see **python-configuration-observability**'s "Route Logs to stdout"
     guidance) — stdout is the intended log pipeline here, not something to
     avoid. The actual mistake in `e.printStackTrace()` / a bare
     `print(traceback)` / `console.log(err)` is **bypassing the structured
     logger**, not writing to stdout as such:
     - An ad hoc print produces unstructured text Datadog can't reliably
       parse into fields (level, service, trace ID, exception type) — it
       becomes an unsearchable, unalertable, trace-uncorrelated log line
       instead of a structured event.
     - The fix routes the exception through the project's configured
       structured logger so it still lands on stdout, correctly formatted:
       ```python
       import logging
       logger = logging.getLogger(__name__)
       try:
           check()
       except CheckError:
           logger.exception("check() failed")  # attaches the full traceback as structured data
       ```
       ```javascript
       try {
         await check();
       } catch (err) {
         logger.error({ err }, "check() failed"); // structured logger (e.g. pino), JSON to stdout
       }
       ```
     - `logger.exception(...)` in Python automatically attaches the current
       exception's traceback to the structured record; a JS structured
       logger's `err` serializer does the equivalent for an `Error` object
       (including its `.stack`). Neither loses the stack trace — both turn it
       into a queryable field instead of unstructured text mixed into
       whatever else writes to stdout.

7. **Clean up resources unconditionally**
   - **Python:** use a context manager (`with open(...) as f:`, or a
     library's own `with SomeClient() as client:`) whenever the object
     supports it — cleanup runs even if the block raises. Fall back to
     `try/finally` only for objects that don't implement the context-manager
     protocol.
   - **JavaScript/TypeScript:** `try { ... } finally { await client.close();
     }` is the reliable, broadly supported pattern — create the resource
     before the `try`, use it inside, and clean it up in `finally` so cleanup
     runs on both the success and failure path. (`using`/`await using`
     declarations — ES2022's explicit resource management — exist in modern
     runtimes and TypeScript 5.2+, but don't assume availability without
     checking the target runtime.)
   - In both languages, the bug to catch in review is cleanup code placed
     *after* the risky call inside the same `try` block with no
     `finally`/context-manager — that cleanup line never runs if the call
     above it raises.

8. **Log an exception exactly once, at the layer that handles it**
   - Catching an exception, logging it, and then re-raising/re-throwing it at
     every layer of a call stack produces one log entry per layer for what is
     really a single failure — noisy during an incident, and misleading if
     someone counts log lines as distinct occurrences.
   - Log at the layer that actually terminates the exception — returns a
     Result, converts it to an HTTP response, retries, or otherwise stops it
     from propagating further — not at every intermediate `except`/`catch`
     that only adds context and re-raises. An intermediate layer wrapping a
     third-party exception (point 4) should do exactly that, without also
     logging it.
   - This is also where a performance concern about exceptions actually has
     merit, and where it doesn't: raising and catching an exception is cheap
     in both Python and JS; what's measurably expensive is materializing and
     formatting a deep stack trace, and logging (formatting plus I/O) is the
     most expensive part of all. That cost multiplies if it happens at every
     layer instead of once. Don't let a general performance worry justify
     avoiding exceptions — reserve the concern for genuinely high-frequency,
     hot-path code raising exceptions for expected outcomes (point 5 already
     rules this out on readability grounds alone), log exactly once per
     failure, and profile the actual runtime if the cost of a specific path
     genuinely needs to be known rather than assuming a cross-language number.

---

## AI decision guidance

When generating exception-handling guidance, keep these principles in mind:

- **Neither Python nor JS/TS enforces exception contracts at compile time** —
  use a typed Result/discriminated-union return for failures callers must
  handle, and reserve raised/thrown exceptions for genuinely exceptional,
  non-branching-worthy failures.
- **Catch as narrowly as the caller's differentiated handling requires** — in
  Python that never means a bare `except:`; in JS it means branching inside
  the single `catch` with `instanceof`.
- **Wrap third-party exceptions at public boundaries**, preserving the cause
  (`from e` / `{ cause: e }`) by default — never let a third-party library's
  exception type become part of your own public contract.
- **"Don't print the stack trace" means "don't bypass the structured
  logger,"** not "avoid stdout" — in this repository's Kubernetes/Datadog
  setup, stdout is correct; unstructured `print()`/`console.log()` output is
  the actual problem.
- **Resource cleanup must run on both the success and failure path** — a
  context manager/`with` (Python) or `try/finally` (JS) is not optional
  polish, it's the difference between reliable cleanup and a resource leak.
- **Pick Result-returning or exception-raising as the default per layer and
  stay consistent** — mixing both within one layer forces callers to handle
  two failure-signaling mechanisms at once; compose fallible steps with
  `map`/`andThen`-style chaining rather than unwrapping and re-checking after
  every call.
- **Log a given failure exactly once**, at the layer that actually handles
  it — not at every intermediate catch-and-rethrow — both for signal quality
  during an incident and because stack-trace materialization/logging is the
  genuinely expensive part of exception handling, not the raise/catch itself.

---

## Success criteria

A strong response should ensure that it:

- **distinguishes "caller must handle this" failures** (Result/discriminated
  union) from **"let it propagate" failures** (raised exception), for
  whichever language is in play,
- **composes fallible steps via chaining** rather than unwrapping and
  branching after every single call,
- **keeps catch clauses as narrow as needed** and never introduces a silent
  catch-all,
- **never lets a third-party exception type appear in a recommended public
  interface**, and preserves the cause chain when wrapping,
- **routes exception logging through the structured logger**, correctly
  reasoned as a format correction, not a destination correction, and **logs
  each failure exactly once**, not at every layer it passes through,
- **shows resource cleanup running unconditionally**, on both the success and
  failure path.

---

## Example prompts for the AI

- "Should this function raise an exception or return a Result if the lookup
  fails?"
- "How do I catch multiple specific exception types in Python without one
  giant catch-all? What about in JavaScript, which only has one catch block?"
- "Someone on the team said not to log exceptions to stdout because it's
  dangerous — is that actually true for us?"
- "How should I wrap this third-party library's exception so it doesn't leak
  into our public API?"
- "Should I catch, log, and re-raise at every layer that touches this
  exception, or just once?"

---

## Related guidance

Use this tool alongside:

- python-domain-error-handling (package `python-core`) — the `Result[T]`
  tagged-union pattern this skill's point 1 builds on for Python, the
  boundary-translation pattern point 1 points to for keeping a layer
  consistently exception-free, and the `from None` concealment guidance
  point 4 narrows against.
- python-configuration-observability (package `python-core`) — structured
  JSON logging to stdout, the mechanics behind point 6's correction.
- architecture-code-duplication-tradeoffs
- architecture-inheritance-coupling-tradeoffs
- architecture-async-exception-handling
- architecture-flexibility-complexity-tradeoffs — the untrusted-caller-code discipline in its hooks-API guidance draws on this skill's exception-design principles
