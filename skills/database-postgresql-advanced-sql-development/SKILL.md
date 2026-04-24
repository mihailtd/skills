---
name: postgresql-advanced-sql-development
description: Guides developers in leveraging advanced SQL features, including Common Table Expressions (CTEs), Window Functions, MERGE statements, JSON processing, and server-side programming in PostgreSQL.
---

# PostgreSQL Advanced SQL Development

This skill helps developers leverage PostgreSQL’s advanced SQL and server-side
capabilities to build maintainable, high-performance database logic.

---

## 1. Common Table Expressions (CTEs) and Recursion

- **Query modularity:** Use `WITH` clauses to create named temporary result sets
  within a larger SQL statement. This improves readability and isolates complex
  calculations from the main query.
- **Recursive hierarchies:** Use `WITH RECURSIVE` to navigate hierarchical data
  structures such as organizational charts, category trees, or nested comments.
  A recursive CTE consists of a base query and a recursive query that references
  the temporary result set, combined with `UNION ALL`.

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
SELECT * FROM employee_tree;
```

---

## 2. Analytics and Window Functions

- **Analytical power:** Use window functions to calculate values over partitions
  without collapsing rows. This preserves detail while producing rankings,
  moving averages, running totals, and other analytics.
- **Abstracting windows:** Use the `WINDOW` clause to define partitioning and
  ordering logic once, then reference it across multiple window functions.
  This avoids duplication and keeps queries maintainable.

Example:

```sql
SELECT
  customer_id,
  amount,
  rank() OVER w AS amount_rank,
  sum(amount) OVER w AS total_amount
FROM sales
WINDOW w AS (PARTITION BY customer_id ORDER BY amount DESC);
```

---

## 3. Advanced Data Manipulation (`MERGE` and UPSERTs)

- **The `MERGE` command:** Use `MERGE` when you need conditional inserts, updates,
  or deletes based on matching rows between a source and target dataset.
- **PostgreSQL 17 enhancements:** On modern PostgreSQL versions, use
  `WHEN NOT MATCHED BY SOURCE THEN` to delete target rows that no longer exist in
  the source, enabling flexible synchronization logic.

Example:

```sql
MERGE INTO target_table t
USING source_table s
ON t.id = s.id
WHEN MATCHED THEN
  UPDATE SET value = s.value
WHEN NOT MATCHED THEN
  INSERT (id, value) VALUES (s.id, s.value)
WHEN NOT MATCHED BY SOURCE THEN
  DELETE;
```

---

## 4. Server-Side Programming (Functions and Procedures)

- **Move logic to the database:** Use UDFs and stored procedures to encapsulate
  complex business logic, automate repetitive tasks, and reduce data transfer.
- **Language flexibility:** PostgreSQL supports SQL, PL/pgSQL, PL/Perl, and
  PL/Python for server-side code.
- **New-style SQL functions:** Use `BEGIN ATOMIC` for standard SQL functions. This
  improves parsing safety and avoids treating the function body as a raw string.
- **Functions vs procedures:** Functions execute within a transaction, while
  stored procedures can manage their own transactions with `CALL`, `COMMIT`, and
  `ROLLBACK`.

Example:

```sql
CREATE FUNCTION calculate_discount(amount numeric)
RETURNS numeric
LANGUAGE sql
AS $$
BEGIN ATOMIC
  SELECT amount * 0.9;
END;
$$;
```

---

## 5. Exploiting JSONB and Advanced Data Types

- **JSONPath navigation:** Use `jsonb_path_query` and related functions to query
  complex `jsonb` documents. JSONPath supports deep navigation, filtering, and
  type-aware extraction.
- **SQL/JSON standard:** PostgreSQL 17 introduces SQL/JSON features such as
  `JSON_EXISTS()`, `JSON_QUERY()`, and `JSON_TABLE`, enabling relational-style
  querying and transformation of JSON data.

Example:

```sql
SELECT jsonb_path_query(data, '$.orders[*] ? (@.status == "shipped")')
FROM order_payloads;
```

Use these capabilities to manage semi-structured data without sacrificing the
power of SQL.
