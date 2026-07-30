---
name: database-postgresql-filter-clause-pivoting

description: Guides the usage of the high-performance FILTER clause with aggregate functions to pivot data swiftly, replacing complex and slower CASE WHEN statements in PostgreSQL.
---

# PostgreSQL FILTER Clause Pivoting

This skill helps developers pivot aggregated data and write partial aggregates
cleanly using PostgreSQL's `FILTER` clause instead of verbose `CASE WHEN`
expressions.

---

## 1. Understand the Concept and Benefits

- **The modern solution:** The `FILTER` clause enables partial aggregates by
  selectively passing rows to individual aggregate functions.
- **Replacing legacy `CASE`:** Before `FILTER`, developers used
  `SUM(CASE WHEN condition THEN 1 ELSE 0 END)` or similar constructs to pivot
  values. `FILTER` is cleaner and easier to read.
- **Performance advantage:** `FILTER` is often faster and more efficient than
  mimicking conditional aggregation with `CASE WHEN ... THEN NULL ... ELSE END`.

---

## 2. Syntax and Implementation

- **How it works:** Append `FILTER (WHERE ...)` directly after an aggregate
  function. It works with `COUNT`, `SUM`, `AVG`, `MAX`, `MIN`, and other
  aggregates.
- **Pivoting example:** Use `FILTER` to create columnized values from row data.

Example:

```sql
SELECT account_id,
       max(phone) FILTER (WHERE phone_type = 'home') AS home_phone,
       max(phone) FILTER (WHERE phone_type = 'work') AS work_phone,
       max(phone) FILTER (WHERE phone_type = 'mobile') AS cell_phone
FROM phone
GROUP BY account_id;
```

- **Conditional averages/counts example:** Compare segments side-by-side.

Example:

```sql
SELECT region,
       avg(production) AS overall_avg,
       avg(production) FILTER (WHERE year < 1990) AS old_avg,
       avg(production) FILTER (WHERE year >= 1990) AS new_avg
FROM t_oil
GROUP BY region;
```

---

## 3. Best Practices: `FILTER` vs. `WHERE`

- **The golden rule:** If all aggregates share the exact same condition, move
  that condition to the main `WHERE` clause.
- **Why:** The `WHERE` clause filters rows before aggregation, reducing the data
  volume read from the table.
- **When to use `FILTER`:** Use it only when other aggregate columns in the same
  `SELECT` list still need the rows that would be filtered out by the main
  `WHERE` clause.

---

## 4. SQL Standards and Portability Warnings

- **The standard:** `FILTER` is part of the ANSI SQL:2003 standard and is
  supported in PostgreSQL since 9.4.
- **Portability limitations:** If you need cross-engine compatibility with
  Oracle, SQL Server, Db2 LUW, or other databases, they may not support `FILTER`.
  In those cases, use the legacy `CASE WHEN` workaround.

---

## Example prompt

- "Use PostgreSQL's FILTER clause to pivot sales totals by region without writing
  `CASE WHEN` expressions."
- "Rewrite this slow `SUM(CASE WHEN ...)` aggregation to use `FILTER` for better
  readability and performance."
