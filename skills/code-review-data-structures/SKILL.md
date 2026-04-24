---
name: code-review-data-structures
description: Guides AI reviewers to evaluate data structure choices for correctness, scalability, performance, and simplicity.
---

# Code Review: Data Structures

This skill helps AI reviewers assess whether data structure choices are appropriate for
real problems by focusing on operations, ordering, uniqueness, relationships,
performance, and maintainability.

---

## When to use this skill

Use this skill when you need to:

- evaluate whether a data structure matches the problem requirements,
- identify inefficient or inappropriate use of arrays, lists, queues, stacks,
  maps, sets, trees, or graphs,
- verify that data size, growth, and operation frequency were considered,
- ensure the implementation avoids forcing a structure into unsupported operations,
- recommend simpler alternatives when complexity is unnecessary.

---

## Outcome

Produce a review that:

- critiques the chosen structure against the actual operations performed,
- checks ordering, uniqueness, and relationship requirements,
- evaluates memory and time complexity for current and projected scale,
- warns about misuse of LIFO/FIFO abstractions,
- recommends simpler, more natural structures when appropriate.

---

## Instructions for the AI

1. **Start from the problem, not the structure**
   - Ask: what operations matter most? search, insert, delete, traversal,
     ordering, min/max, or hierarchy?
   - Validate the reviewer’s conclusion using the top 20% of operations that
     drive most of the runtime.

2. **Evaluate the chosen structure against operations**
   - If frequent lookup is required, prefer hash maps, indexed arrays, or trees
     over linear scans.
   - If insert/delete frequency is high, verify the structure supports cheap
     updates (linked list, deque, or balanced tree rather than array shifts).
   - If ordering matters, verify the use of FIFO, LIFO, sorted structures, or
     explicit order-preserving collections.

3. **Check ordering and uniqueness requirements**
   - Confirm queues are used for FIFO behavior and stacks for LIFO behavior.
   - Confirm sets/maps are used when uniqueness is required.
   - Warn if a list is used to enforce uniqueness, as that often hides an
     inefficient choice.

4. **Validate element relationships**
   - If data is hierarchical, favor tree-like structures instead of flat arrays.
   - If elements belong together, avoid parallel arrays unless the relationship is
     absolutely simple and stable.
   - Prefer a single composite structure (map of records, object list) over
     separate synced arrays for related data.

5. **Analyze performance and scalability**
   - Compare the structure’s average and worst-case complexity to the likely
     problem size.
   - Ask whether the chosen structure still works well if the dataset grows by
     one or two orders of magnitude.
   - Call out O(n) searches or O(n) shifts where a more efficient structure is
     clearly needed.

6. **Favor simplicity when multiple fits exist**
   - If two structures both satisfy the requirements, recommend the simpler one.
   - Prefer structures that make the code easier to understand and maintain.
   - Resist complex structures unless the performance need is proven.

7. **Be alert for common anti-patterns**
   - Arrays used as queues with repeated shifting.
   - Hash map keys that are mutated after insertion.
   - Parallel arrays for related data instead of a single associative structure.
   - Trees used where a hash map or simple list would suffice.
   - Overly clever structures for tiny or stable datasets.

---

## Decision points and guidance

- **Does the code match the operations?** If the main operations are search-heavy,
  ensure the structure supports fast lookup.
- **Is ordering actually required?** If yes, choose a structure that preserves or
  enforces the needed order naturally.
- **Is uniqueness required?** If yes, use a set or map instead of manually checking duplicates.
- **Are related pieces forced into parallel structures?** If yes, suggest a more
  cohesive representation.
- **Is the structure scalable?** If the dataset may grow, favor structures with
  better asymptotic performance for the hot-path operations.
- **Is a simpler structure sufficient?** If so, avoid needlessly complex data models.

---

## Quality criteria

A strong data structure review should confirm that the implementation is:

- **operation-appropriate:** the structure fits the actual workload,
- **ordering-aware:** order semantics match the problem requirements,
- **unique-aware:** uniqueness is modeled by the right abstraction,
- **relationship-aligned:** data relationships are represented naturally,
- **scalable:** performance and memory match current and likely future size,
- **simple:** the choice is no more complex than necessary,
- **clear:** the structure is easy to reason about and maintain,
- **correct:** the structure is not being forced into unsupported operations.

---

## Review checklist

Ask the following questions during the review:

- [ ] Are the top operations on the dataset explicitly identified?
- [ ] Is the chosen data structure appropriate for those operations?
- [ ] Does the implementation preserve required ordering or prioritization?
- [ ] Does it enforce uniqueness where needed?
- [ ] Does the data model reflect linear vs. hierarchical relationships?
- [ ] Would a simpler structure have been safer and easier to maintain?
- [ ] Are there hidden O(n) costs being introduced by the chosen structure?
- [ ] Is the structure sized and chosen with projected growth in mind?
- [ ] Are queues and stacks used for the right semantics (FIFO/LIFO)?
- [ ] Is the implementation free of parallel arrays or fragile synchronization?

---

## Example prompts

- "Review the data structures in this code and tell me whether they match the
  priority operations."
- "Does this implementation use the right structure for search-heavy workloads?"
- "Evaluate whether the chosen collection is scalable and simple enough for the
  problem." 

---

## Related guidance

This skill complements:

- code-review-code-structure
- architecture-risk-management
- architecture-building-with-failure-in-mind
- business-analysis-scope