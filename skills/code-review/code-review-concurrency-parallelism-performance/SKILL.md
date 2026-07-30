---
name: code-review-concurrency-parallelism-performance
description: Guides AI reviewers to evaluate concurrency, parallelism, and performance trade-offs in code.
---

# Code Review: Concurrency, Parallelism, and Performance

This skill helps AI reviewers assess whether concurrent and parallel code is correct,
scalable, and justified by the workload. It focuses on safety, liveness,
performance cost, and architecture-level trade-offs.

---

## When to use this skill

Use this skill when you need to:

- review concurrent or parallel implementations,
- validate thread safety and process isolation,
- verify that parallelism is justified for the workload,
- assess the impact of shared state and locking,
- ensure performance trade-offs are realistic and well-managed.

---

## Outcome

Produce a review that:

- identifies correctness issues in concurrent code,
- evaluates the cost of parallel execution,
- flags hidden contention, deadlock risk, and data locality problems,
- recommends simpler, safer approaches when possible,
- confirms that performance design is driven by actual requirements.

---

## Instructions for the AI

1. **Distinguish concurrency from parallelism**
   - Treat concurrency as structuring interleaving execution and shared-resource access.
   - Treat parallelism as simultaneous execution for throughput/latency gains.
   - Verify that the code uses the right abstraction for the problem.

2. **Check correctness: safety and liveness**
   - Safety: ensure shared mutable state is protected and invariants are preserved.
   - Liveness: ensure no deadlocks, starvation, or unfair scheduling is introduced.
   - Confirm resources are acquired in a consistent order and released reliably.

3. **Prefer functional-lite thread safety**
   - Favor immutability, stateless design, and referential transparency.
   - If shared state is necessary, require context managers or equivalent scoped locking.
   - Avoid manual acquire()/release() patterns that can leak locks on exceptions.

4. **Validate transaction and integrity assumptions**
   - Verify ACID expectations for operations that span multiple updates.
   - Confirm atomicity, consistency, isolation, and durability requirements are realistic.
   - Call out cases where lower isolation or optimistic concurrency would improve throughput safely.

5. **Assess parallelization costs**
   - Evaluate split/merge overhead, communication latency, and load imbalance.
   - Use Amdahl’s Law to judge whether additional parallel resources are likely to help.
   - Check whether the sequential portion of the workload dominates execution time.

6. **Evaluate data locality and resource usage**
   - Ensure computation is performed near the data to minimize transfer overhead.
   - Flag excessive data movement between processes, threads, or nodes.
   - Review memory and cache usage for long-running concurrent tasks.

7. **Apply Python-specific guidance when relevant**
   - For compute-bound work in Python, prefer multiprocessing over threading.
   - For I/O-bound work, threading or async may be appropriate.
   - Confirm the Global Interpreter Lock (GIL) is not being ignored in performance claims.

8. **Demand measurable justification**
   - Require that parallelism is justified by actual data size, not by theoretical asymptotic advantage alone.
   - Prefer simple sequential implementations until profiling proves the need for parallelism.
   - Flag premature optimization and overly complex orchestration layers.

---

## Decision points and guidance

- **Is this a correctness issue or a performance issue?** If both, fix correctness first.
- **Does shared state exist?** If yes, ensure it is carefully isolated or locked.
- **Can the work be made stateless or immutable?** If yes, prefer that.
- **Is the parallelization overhead justified?** If not, recommend a simpler sequential design.
- **Are locks managed safely?** If not, require structured locking or a different concurrency model.
- **Does the implementation respect the runtime environment?** For Python, ensure the GIL, process costs, and IPC overhead are accounted for.

---

## Quality criteria

A strong concurrency and performance review should verify that the code is:

- **correct:** no unsafe races, deadlocks, or inconsistent state,
- **live:** progress is guaranteed and tasks are not starved,
- **justified:** parallelism is driven by workload and data size,
- **efficient:** overheads are minimized and data locality considered,
- **simple:** the design avoids unnecessary complexity,
- **measurable:** performance claims are backed by actual metrics or reasonable projections,
- **environment-aware:** implementation matches Python or platform-specific constraints,
- **maintainable:** concurrent behavior is explicit and easy to reason about.

---

## Review checklist

Use these questions during the review:

- [ ] Does the implementation correctly separate concurrency from parallelism?
- [ ] Is shared mutable state protected and managed safely?
- [ ] Are locks used with scoped context management, not manual release patterns?
- [ ] Are deadlock and starvation risks clearly mitigated?
- [ ] Could the design be simplified by making more code immutable or stateless?
- [ ] Is the parallelism overhead justified by the workload size and structure?
- [ ] Are data locality and communication costs minimized?
- [ ] Are ACID-like transactional assumptions explicitly stated and validated?
- [ ] For Python, is threading used only for I/O-bound cases and multiprocessing for CPU-bound cases?
- [ ] Has premature optimization been avoided in favor of clear design?

---

## Example prompts

- "Review this concurrent implementation for race conditions and performance trade-offs."
- "Is this parallelization worth the overhead, or should the code stay sequential?"
- "Does this Python code actually benefit from threads, or is the GIL limiting it?"

---

## Related guidance

This skill complements:

- code-review-code-structure
- code-review-data-structures
- code-review-detect-bad-design
- architecture-risk-management
- architecture-building-with-failure-in-mind