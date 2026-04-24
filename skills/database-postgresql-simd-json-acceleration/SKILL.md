---
name: database-postgresql-simd-json-acceleration

description: Guides the implementation of SIMD (Single Instruction, Multiple Data) acceleration in PostgreSQL 16 to massively optimize ASCII string and JSON data processing.
---

# PostgreSQL SIMD JSON Acceleration

This skill helps developers leverage PostgreSQL 16's SIMD acceleration for
high-volume ASCII and JSON string processing, improving query throughput and
CPU efficiency.

---

## 1. Understand SIMD Acceleration in PostgreSQL 16

- **The concept:** PostgreSQL 16 introduced SIMD acceleration to speed up string
  and JSON processing by applying single instructions to multiple data elements
  simultaneously.
- **The benefit:** SIMD improves performance for data-intensive workloads,
  especially when processing ASCII text or JSON values, by reducing CPU cycles
  and improving throughput.
- **When it applies:** PostgreSQL applies SIMD automatically where the planner
  detects suitable operators and data patterns.

---

## 2. Identifying Performance Bottlenecks

- **Assess the workload:** Start by identifying queries that perform heavy JSON
  extraction, filtering, or transformation on large tables.
- **Use execution plans:** Run `EXPLAIN ANALYZE` to inspect the query plan and
  identify the slowest operators and the amount of data processed.
- **Target high-cardinality JSON paths:** Look for repeated JSON operators,
  `jsonb_array_elements_text()`, `->>`, or string comparisons inside large scans.

---

## 3. Restructuring Queries for SIMD

- **Automatic application:** PostgreSQL 16 applies SIMD automatically when the
  query shape and operators are amenable to vectorized execution.
- **Focus on structure:** The developer should focus on writing clean SQL that
  eliminates unnecessary work and narrows the dataset early.
- **Narrow the dataset early:** Use filtering in subqueries or CTEs to reduce the
  number of rows before heavy JSON extraction or string processing occurs.

Example:

```sql
SELECT order_id, item
FROM (
  SELECT order_id, jsonb_array_elements_text(order_details -> 'items') AS item
  FROM orders
  WHERE order_details ->> 'status' = 'completed'
) AS items_processed
WHERE item IS NOT NULL;
```

- **Why it helps:** This structure pushes row reduction before the JSON array
  expansion and makes the work more likely to be processed with SIMD-friendly
  operators.

---

## 4. Implementation Example

- **Original query:** A straightforward JSON extraction query may process more
  rows than necessary and invoke expensive operators before filtering.
- **SIMD-optimized query:** Use a focused subquery or CTE to restrict the input
  set and then perform JSON functions on the reduced row set.

Original query:

```sql
SELECT order_id, jsonb_array_elements_text(order_details -> 'items') AS item
FROM orders
WHERE (order_details ->> 'status') = 'completed';
```

SIMD-optimized subquery:

```sql
SELECT order_id, item
FROM (
  SELECT order_id, jsonb_array_elements_text(order_details -> 'items') AS item
  FROM orders
  WHERE (order_details ->> 'status') = 'completed'
) AS items_processed
WHERE item IS NOT NULL;
```

- **Verification:** Use `EXPLAIN ANALYZE` on both versions to compare actual
  runtime, buffer usage, and whether the plan contains SIMD-enabled operators.

---

## 5. Verify and Compare Performance

- **Measure before and after:** Always compare both query versions with
  `EXPLAIN ANALYZE` to ensure the restructuring actually improves performance.
- **Check for SIMD support:** PostgreSQL's execution plan may include notes about
  vectorized string operations on supported platforms.
- **Iterate:** If the plan still shows expensive JSON processing, try adding
  more selective filters earlier in the query or using indexes on JSON-derived
  values.

Example:

```sql
EXPLAIN ANALYZE
SELECT order_id, item
FROM (
  SELECT order_id, jsonb_array_elements_text(order_details -> 'items') AS item
  FROM orders
  WHERE (order_details ->> 'status') = 'completed'
) AS items_processed
WHERE item IS NOT NULL;
```

---

## Example prompt

- "Explain how PostgreSQL 16 uses SIMD to accelerate JSON string processing and
  how I can restructure a query to take advantage of it."
- "Show me how to compare a JSON extraction query before and after SIMD-friendly
  rewriting using `EXPLAIN ANALYZE`."
- "Help me optimize a large `jsonb_array_elements_text()` query by filtering rows
  earlier in the plan."