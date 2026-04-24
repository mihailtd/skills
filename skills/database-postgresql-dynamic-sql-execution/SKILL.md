---
name: postgresql-dynamic-sql-execution
description: Guides the secure and efficient execution of dynamic SQL within PostgreSQL functions to overcome traditional planner limitations, avoid cached plan performance degradation, and prevent SQL injection.
---

# PostgreSQL Dynamic SQL Execution

This skill helps developers use dynamic SQL in PostgreSQL functions safely and
efficiently, especially when query shape or cardinality varies and cached plans
can degrade performance.

---

## 1. Understand the Purpose of Dynamic SQL

- **The concept:** Dynamic SQL is when a SQL statement is constructed as a text
  string and then executed with the `EXECUTE` command in PL/pgSQL.
- **The cached plan trap:** Standard PL/pgSQL functions may cache the execution
  plan from the first call in a session. If later calls have very different
  parameter values, the cached plan can become suboptimal and hurt performance.
- **The dynamic solution:** Dynamic SQL forces PostgreSQL to re-evaluate the
  execution plan for each call, which may add a small planning overhead but can
  dramatically improve performance for variable query shapes.

---

## 2. Implementing and Aiding the Optimizer

- **Function encapsulation:** Wrap dynamic SQL inside a function and build the
  query text based on the input parameters.
- **Optimizer nudging:** Use dynamic SQL to steer the planner toward a better
  plan tailored to the exact runtime values.
- **FDW optimization:** When working with Foreign Data Wrappers, dynamic SQL can
  narrow remote queries by identifying needed keys locally and passing explicit
  IDs to the remote side.

Example:

```sql
CREATE FUNCTION get_orders(customer_id int, period text)
RETURNS TABLE(order_id int, total numeric) AS $$
BEGIN
  RETURN QUERY EXECUTE format(
    'SELECT order_id, total FROM orders WHERE customer_id = %L AND period = %L',
    customer_id, period
  );
END;
$$ LANGUAGE plpgsql;
```

---

## 3. Strict Security and SQL Injection Prevention

- **The danger of concatenation:** Never build SQL by blindly concatenating user
  input with `||`; this is a common SQL injection risk.
- **The `format()` solution:** Use `format()` to build SQL safely, and combine it
  with `EXECUTE ... USING` for runtime values.
- **Quoting literals:** When you must inject values into the SQL text directly,
  sanitize them with `quote_literal()` for values and `quote_ident()` for
  identifiers.

Example:

```sql
EXECUTE format(
  'SELECT * FROM %I WHERE id = $1',
  quote_ident(table_name)
)
USING id_value;
```

---

## 4. Best Practices

- Use dynamic SQL only when the query structure changes based on parameters.
- Keep dynamic SQL logic isolated in database functions.
- Prefer `EXECUTE ... USING` over direct value interpolation.
- Validate or quote identifiers with `quote_ident()`.
- Use `quote_literal()` for values that must be embedded directly into SQL text.
- Monitor `EXPLAIN` plans to ensure the dynamic SQL provides better performance
  than a generic cached-plan alternative.

---

## Example prompt

- "Write a secure PL/pgSQL function that builds and executes a dynamic SQL
  query based on the requested table name and filter value."
- "Explain how dynamic SQL avoids cached plan issues in PostgreSQL functions and
  show how to prevent SQL injection."
