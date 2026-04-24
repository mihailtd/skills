---
name: postgresql-lateral-joins
description: Guides the usage of LATERAL JOINs to iterate subqueries successfully row-by-row, enabling parameterized subqueries and loop-like behavior in PostgreSQL.
---

# PostgreSQL LATERAL Joins

This skill helps developers use `LATERAL` joins in PostgreSQL to execute a
subquery once per row of the outer query, enabling parameterized subqueries and
loop-like behavior inside a SQL statement.

---

## 1. Understand the Concept of LATERAL JOINs

- **The concept:** A `LATERAL` join allows a table expression to reference
  columns from preceding tables in the `FROM` list.
- **The execution order:** The subquery is evaluated for each row of the outer
  table, using that row's values as parameters.
- **The loop analogy:** `LATERAL` behaves like a loop in SQL, executing the
  subquery once per outer row.

---

## 2. Highlight Key Benefits and Use Cases

- **Accessing subquery data:** Unlike `EXISTS`, a `LATERAL` join can return
  actual values from the subquery and make them available to the outer query.
- **Parameterized joins:** Use `LATERAL` when the subquery depends on columns
  from the main table.
- **Processing lists and functions:** Ideal for joining each row to a function
  result or a generated series specific to that row.

---

## 3. Syntax and Implementation Examples

- **Relational query example:** Join a table against a correlated subquery.

Example:

```sql
SELECT u.username, q.*
FROM users u
JOIN LATERAL (
  SELECT author, title, likes
  FROM posts p
  WHERE u.pk = p.author
    AND likes > 2
) AS q ON true;
```

- **Series/function example:** Use `LATERAL` to generate row-specific results.

Example:

```sql
SELECT x, z.values
FROM generate_series(1, 4) AS x,
LATERAL (
  SELECT array_agg(y) AS values
  FROM generate_series(1, x) AS y
) AS z;
```

---

## 4. Adoption and Compatibility

- **Version support:** `LATERAL` has been supported in PostgreSQL since 9.3.
- **Underappreciated feature:** It is a powerful tool for advanced SQL but often
  underused by developers.

---

## Example prompt

- "Use a LATERAL join to select each user and their top 3 posts based on likes."
- "Demonstrate how to pass the outer row into a subquery using LATERAL in PostgreSQL."
