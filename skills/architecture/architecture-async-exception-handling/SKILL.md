---
name: architecture-async-exception-handling

description: Instructs the AI assistant on why exceptions can be silently lost in fire-and-forget async/concurrent code in Python and JavaScript — an asyncio Task or ThreadPoolExecutor Future whose result is never consumed, or a Promise never awaited/caught — and how to prevent it — always consume the result, use TaskGroup/Promise.allSettled to aggregate multiple outcomes without losing individual failures, and treat a global handler as a safety net, not a primary strategy.
---

# Async Exception Handling Instructions

When supporting concurrent or async code design in Python or JavaScript, use
this tool to explain why fire-and-forget concurrent work risks losing
exceptions silently, and how each language's tooling (Python's `TaskGroup`,
JavaScript's `Promise.allSettled`) prevents that when multiple concurrent
operations' outcomes all matter.

---

## Purpose

This tool helps the AI assistant by:

- explaining why creating concurrent work and not consuming its outcome loses
  exceptions silently in both Python (an unawaited `asyncio.Task`, an
  unconsumed `ThreadPoolExecutor` `Future`) and JavaScript (a `Promise` never
  `await`ed or given a `.catch()`),
  recommending that a task/future/promise's outcome always be consumed — or,
  when genuinely fire-and-forget, given an explicit error callback rather than
  none at all,
- distinguishing fail-fast concurrent aggregation (`asyncio.gather`,
  `Promise.all` — stop and report on the first failure) from
  collect-everything aggregation (`asyncio.TaskGroup`, `Promise.allSettled` —
  report every outcome), and picking between them based on whether the caller
  needs to know about every failure or just the first,
- treating a global exception/rejection handler as a safety net for bugs, not
  a substitute for consuming results directly — routine firing of the global
  handler is a signal to fix the underlying pattern, not something to tune
  around.

---

## Expected outcome

As the AI, your response should help produce concurrent/async code that:

- never creates a task, future, or promise without a defined way its failure
  will be observed — consumed directly, or given an explicit error callback,
- picks fail-fast vs. collect-all aggregation deliberately, based on whether
  the caller needs every concurrent operation's outcome or just to know that
  something failed,
- logs any exception surfaced from concurrent work through the project's
  structured logger (see **architecture-exception-design-and-anti-patterns**
  point 5), not silently or via an unstructured print,
- treats a global handler firing as evidence of a gap to close in the code
  that created the unconsumed work, not as the intended error-handling path.

---

## Instructions for the AI

1. **Explain why fire-and-forget concurrent code loses exceptions silently**
   - **Python:** `asyncio.create_task(coro())` schedules work and returns
     immediately — if the task raises and nothing ever `await`s it or calls
     `.result()`/`.exception()` on it, the exception only surfaces (as a
     "Task exception was never retrieved" warning) when the task is
     garbage-collected, which can happen much later or not at all during a
     long-running process. `ThreadPoolExecutor.submit(fn)` has the identical
     shape of problem: the returned `Future`'s exception is only raised when
     `.result()` is called.
   - **JavaScript:** a `Promise` that rejects and is never `await`ed or given
     a `.catch()` produces an unhandled promise rejection — modern Node.js
     treats this as fatal by default (the process exits), which is *safer*
     than Python's silent case, but still means the failure was never handled
     by application logic, only crashed the process.
   - The lesson is the same in both: creating concurrent work without a plan
     for observing its failure is the async equivalent of fire-and-forget
     submission without a `Future` — always decide up front how a concurrent
     operation's failure will be seen, even if the plan is "log it and move
     on."
   - One Java-specific pain point does *not* carry over: wrapping a
     synchronous, exception-throwing call inside `CompletableFuture.runAsync`
     requires manually catching the exception and calling
     `completeExceptionally(...)`, or the exception gets wrapped in a
     `CompletionException` and surfaces as a stack trace spanning many
     concurrency-library frames before reaching the actual cause. Python's
     `asyncio.to_thread(fn)`/`loop.run_in_executor(...)` and any `async def`
     function, and JS's `async function`s and Promises, capture a
     raised/thrown exception directly into the returned awaitable's failure
     state automatically — `await`ing it re-raises the original exception
     with no manual completion step and no extra wrapping layer to explain
     away in a log. This is a case where the source material's workaround
     doesn't need translating — the underlying problem doesn't exist in
     either language's async model.

2. **Always consume the result — or state explicitly why not**
   ```python
   # risky: exception from do_work() is only surfaced if/when the task is GC'd
   asyncio.create_task(do_work())

   # correct: the caller observes success or failure
   task = asyncio.create_task(do_work())
   try:
       await task
   except Exception:
       logger.exception("do_work() failed")
   ```
   ```javascript
   // risky: unhandled rejection if doWork() fails
   doWork();

   // correct: failure is observed and logged
   doWork().catch((err) => logger.error({ err }, "doWork() failed"));
   ```
   - If work genuinely needs to run detached from anything awaiting it (a true
     background job), that's still not license to skip error handling —
     attach a `.catch`/error callback that logs, so a failure is visible in
     the structured logger instead of disappearing.

3. **Choose the right aggregation tool for multiple concurrent operations**
   - **Python:** `asyncio.gather(*tasks)` (without `return_exceptions=True`)
     cancels the remaining tasks and raises on the *first* failure — visibility
     into whether any of the other tasks also failed is lost.
     `asyncio.TaskGroup` (3.11+) is the structured-concurrency alternative: it
     waits for all tasks, cancels siblings on a failure, and raises an
     `ExceptionGroup` containing *every* exception that occurred — prefer it
     whenever the caller needs to know about every failure, not just that
     something failed.
   - **JavaScript:** `Promise.all([...])` rejects as soon as the first promise
     rejects, discarding the outcomes of the others — the same fail-fast
     tradeoff as `gather`. `Promise.allSettled([...])` always resolves, with
     each entry reporting `{status: 'fulfilled', value}` or `{status:
     'rejected', reason}` — use it when the caller needs every outcome (e.g.
     "process these 20 independent items and report which ones failed"), not
     just the first failure.
   - Decision rule for both: if any individual failure should stop the whole
     operation immediately (genuinely all-or-nothing), fail-fast
     (`gather`/`Promise.all`) is correct. If the operations are independent
     and the caller needs a full picture of what succeeded and failed, use
     the collect-everything variant (`TaskGroup`/`Promise.allSettled`).

4. **Use a global handler as a safety net, not a primary strategy**
   - **Python:** `loop.set_exception_handler(...)` (asyncio) catches whatever
     wasn't otherwise retrieved — configure it to log loudly through the
     structured logger rather than silently discard, but treat anything that
     reaches it as a bug in code that should have consumed the task's result
     directly, not as the intended path.
   - **JavaScript:** `process.on("unhandledRejection", ...)` (Node) is the
     equivalent backstop — log the rejection with full context, and seriously
     consider whether the process should exit afterward (per Node's own
     guidance), since an unhandled rejection usually means the application
     reached a state it didn't anticipate.
   - In both cases, a global handler firing routinely — not just for a
     genuine one-off bug — signals that fire-and-forget work throughout the
     codebase isn't consuming its results; treat that as a prompt to audit
     for the pattern in point 2, not as a permanent fix in itself.

---

## AI decision guidance

When generating async/concurrent exception-handling guidance, keep these
principles in mind:

- **Fire-and-forget concurrent work is a silent-failure risk in both
  languages** — always plan for how its outcome will be observed before
  creating it.
- **Prefer the structured-concurrency, collect-every-outcome tool**
  (`TaskGroup`, `Promise.allSettled`) whenever a caller genuinely needs to
  know about every concurrent operation's result, not just the first failure.
- **A global exception/rejection handler is a safety net for bugs**, not a
  substitute for consuming task/future/promise results directly — its routine
  firing is a signal to fix, not tune around.

---

## Success criteria

A strong response should ensure that it:

- **flags fire-and-forget concurrent code that never consumes its result** as
  a silent-failure risk,
- **picks fail-fast vs. collect-all aggregation deliberately**, based on
  whether the caller needs every outcome,
- **treats a global handler as a backstop**, not a primary error-handling
  strategy,
- **routes any caught async failure through the structured logger**,
  consistent with **architecture-exception-design-and-anti-patterns**.

---

## Example prompts for the AI

- "I fired off a background task and it seems to have failed silently — why?"
- "Should I use `Promise.all` or `Promise.allSettled` here?"
- "What's the Python equivalent of a global uncaught-exception handler for
  async code?"

---

## Related guidance

Use this tool alongside:

- architecture-exception-design-and-anti-patterns — the synchronous
  exception-design half of this same source material.
- python-asyncio-concurrent-web-requests (package `python-core`) — the
  underlying asyncio concurrency mechanics this skill's exception-handling
  guidance applies to.
- python-asyncio-synchronization (package `python-core`)
- python-configuration-observability (package `python-core`) — the structured
  logger these examples route failures through.
