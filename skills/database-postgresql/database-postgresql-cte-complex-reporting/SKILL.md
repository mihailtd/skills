---
name: database-postgresql-cte-complex-reporting

description: Guides the usage of Common Table Expressions (CTEs) and the WITH clause to organize massively complex reporting queries, including recursion and materialization optimization.
---

# PostgreSQL CTE Complex Reporting

This skill helps developers simplify massive reporting queries by breaking them
into readable, modular components using Common Table Expressions (CTEs).

---

## 1. Understand CTEs and Query Modularity

- **The concept:** CTEs introduce elegance and modularity to SQL by creating
  named temporary result sets within the context of a larger query.
- **Readability and maintenance:** Unlike deeply nested subqueries, CTEs provide
  abstraction and allow complex logic to be broken into manageable parts.
- **Eliminating redundancy:** Define a complex subquery once and reference it
  multiple times throughout the query to reduce duplication and ensure
  consistent results.

Example:

```sql
WITH base_sales AS (
  SELECT customer_id, order_id, amount
  FROM orders
  WHERE order_date >= '2025-01-01'
),
ranked_sales AS (
  SELECT customer_id, amount,
         rank() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS sales_rank
  FROM base_sales
)
SELECT *
FROM ranked_sales
WHERE sales_rank <= 3;
```

---

## 2. Syntax and Chaining Multiple CTEs

- **The syntax:** Use `WITH CTEName AS ( ... ) SELECT * FROM CTEName;`.
- **Chaining CTEs:** For complex reports, chain multiple CTEs separated by commas
  after a single `WITH` keyword.
- **Step-by-step aggregation:** Create a chain of CTEs that incrementally
  aggregates, cleans, and enriches data before the final query.

Example:

```sql
WITH
  sales_by_customer AS (
    SELECT customer_id, SUM(amount) AS total_sales
    FROM orders
    GROUP BY customer_id
  ),
  active_customers AS (
    SELECT customer_id
    FROM customers
    WHERE active = true
  )
SELECT a.customer_id, s.total_sales
FROM active_customers a
JOIN sales_by_customer s USING (customer_id);
```

---

## 3. Performance and Materialization (PostgreSQL 12+)

- **The inlining shift:** PostgreSQL 12 and later automatically inline CTEs
  referenced only once and without side effects.
- **Optimizer flexibility:** This allows the planner to push down filters and
  generate a better execution plan than the old optimization fence behavior.
- **Forcing materialization:** Use `MATERIALIZED` to force the CTE to be
  evaluated once and stored in memory as a temporary result.
- **Forcing inlining:** Use `NOT MATERIALIZED` to ensure the CTE is inlined into
  the main execution plan.

Example:

```sql
WITH fast_lookup AS MATERIALIZED (
  SELECT id, value
  FROM lookup_table
)
SELECT t.*, f.value
FROM transactional_data t
JOIN fast_lookup f ON t.lookup_id = f.id;
```

---

## 4. Handling Hierarchies with `WITH RECURSIVE`

- **Hierarchical data:** Use `WITH RECURSIVE` for organizational charts,
  category trees, or graph-style relationship traversal.
- **The structure:** A recursive CTE has a base query, a `UNION` / `UNION ALL`
  operator, and a recursive query referencing the CTE itself.
- **Infinite loops:** Guard against infinite recursion, especially in messy
  or cyclic data.
- **`UNION` vs `UNION ALL`:** `UNION` can help prevent infinite loops by
dropping duplicate rows, while `UNION ALL` is faster but requires cleaner data.

Example:

```sql
WITH RECURSIVE employee_tree AS (
  SELECT id, manager_id, name
  FROM employees
  WHERE manager_id IS NULL
  UNION ALL
  SELECT e.id, e.manager_id, e.name
  FROM employees e
  JOIN employee_tree t ON e.manager_id = t.id
)
SELECT *
FROM employee_tree;
```

---

## 5. Best Practices

- Use CTEs to isolate logical steps, not as a substitute for poor schema design.
- Name CTEs clearly to express their purpose.
- Chain CTEs to build complex reports in layers of transformation.
- Test materialization behavior with `EXPLAIN` and adjust `MATERIALIZED`
  hints only when needed.
- Keep recursive queries bounded and indexed on the parent-child key.
