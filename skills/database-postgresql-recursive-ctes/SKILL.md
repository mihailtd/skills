---
name: postgresql-recursive-ctes
description: Guides the implementation of Recursive Common Table Expressions (CTEs) to efficiently query hierarchical data structures, prevent infinite loops, and navigate trees in PostgreSQL.
---

# PostgreSQL Recursive CTEs

This skill helps developers implement recursive queries in PostgreSQL for
hierarchical data such as organizational charts, categories, and comment trees.

---

## 1. Understand the Purpose of Recursive CTEs

- **The concept:** A recursive CTE using `WITH RECURSIVE` allows a query to
  reference its own output.
- **Best use cases:** Ideal for hierarchies with unknown depth, such as employee
  management chains, nested categories, or threaded comments.
- **The benefit:** It replaces manual recursion or cursor-based iteration with
  declarative SQL, keeping the code clean and maintainable.

---

## 2. The Three-Part Structure

A recursive CTE requires three parts:

- **Base query:** the starting point, such as top-level rows.
- **Operator:** `UNION` or `UNION ALL` joins the base and recursive parts.
- **Recursive query:** references the CTE itself to walk the hierarchy.

Example:

```sql
WITH RECURSIVE employee_hierarchy AS (
  -- Base Query
  SELECT employee_id, first_name, manager_id
  FROM employees
  WHERE manager_id IS NULL

  UNION ALL

  -- Recursive Query
  SELECT e.employee_id, e.first_name, e.manager_id
  FROM employees e
  JOIN employee_hierarchy eh ON eh.employee_id = e.manager_id
)
SELECT *
FROM employee_hierarchy;
```

---

## 3. Preventing Infinite Loops (Crucial Warning)

- **The danger:** Recursive CTEs can run forever if the recursion does not terminate,
  especially when the data contains cycles.
- **`UNION` vs `UNION ALL`:** `UNION ALL` is faster but allows duplicates and may
  continue indefinitely on cyclic data. `UNION` removes duplicates and can
  protect against simple loops by stopping once rows repeat.
- **Indexing:** Ensure the parent-child key used in the recursive join is indexed
  for performance.

---

## 4. Practical Tips

- Start with a simple base query and test it before adding recursion.
- Choose `UNION ALL` for performance when the data is known to be acyclic.
- Use `UNION` when the data may contain duplicates or circular references.
- Add a depth counter or stop condition if you need to enforce a maximum level.

Example with depth:

```sql
WITH RECURSIVE employee_hierarchy AS (
  SELECT employee_id, first_name, manager_id, 1 AS depth
  FROM employees
  WHERE manager_id IS NULL

  UNION ALL

  SELECT e.employee_id, e.first_name, e.manager_id, eh.depth + 1
  FROM employees e
  JOIN employee_hierarchy eh ON eh.employee_id = e.manager_id
  WHERE eh.depth < 10
)
SELECT *
FROM employee_hierarchy;
```

---

## 5. When to Use Recursive CTEs

- Traversing parent-child hierarchies without a priori depth limits.
- Generating ancestor/descendant lists for tree structures.
- Expanding nested categories or organizational reporting chains.
- Avoiding procedural loops in the client application.

Use recursive CTEs when the data is naturally hierarchical and the query needs to
explore multiple levels dynamically.
