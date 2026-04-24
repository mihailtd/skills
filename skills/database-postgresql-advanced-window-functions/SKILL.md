---
name: postgresql-advanced-window-functions
description: Guides the accurate application of advanced Window Functions, frame clauses, and sliding windows to perform complex analytics and calculate moving averages in PostgreSQL.
---

# PostgreSQL Advanced Window Functions

This skill helps developers use PostgreSQL window functions precisely for complex
analytics, moving averages, ranking, and time-series calculations.

---

## 1. The Core of Window Functions

- **The concept:** A window function calculates values across a set of rows
  related to the current row, similar to an aggregate, but it preserves the
  original rows rather than collapsing them.
- **The `OVER` clause:** Every window function requires an `OVER` clause; without
  it, PostgreSQL cannot evaluate the window function.
- **Partitioning data:** Use `PARTITION BY` inside `OVER` to divide data into
  subsets, allowing calculations to reset for each group.

Example:

```sql
SELECT
  country,
  year,
  revenue,
  avg(revenue) OVER (PARTITION BY country) AS avg_country_revenue
FROM sales;
```

---

## 2. Sliding Windows for Moving Averages

- **Defining the frame:** A moving average requires a frame that slides along the
  ordered rows.
- **The `ORDER BY` requirement:** A sliding window must include an `ORDER BY`
  clause; without it, row order is undefined and results may be incorrect.
- **Frame syntax:** Use `ROWS BETWEEN start_point AND end_point` to define the
  window boundaries.

Example:

```sql
SELECT
  date,
  revenue,
  avg(revenue) OVER (
    ORDER BY date
    ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
  ) AS moving_avg
FROM daily_sales;
```

- **Running totals:** Use `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
  for cumulative sums.

Example:

```sql
SELECT
  date,
  revenue,
  sum(revenue) OVER (
    ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total
FROM daily_sales;
```

---

## 3. The Subtle Difference Between `ROWS` and `RANGE`

- **`ROWS` (physical rows):** Operates on a fixed number of rows relative to the
  current row.
- **`RANGE` (logical groups):** Treats rows with equal `ORDER BY` values as a
  group, which can affect duplicate values.
- **Managing duplicates:** Use `EXCLUDE TIES` to remove duplicate boundary rows,
  or `EXCLUDE GROUP` to remove the entire set of matching rows.

Example:

```sql
SELECT
  date,
  value,
  avg(value) OVER (
    ORDER BY date
    RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
  ) AS weekly_avg
FROM time_series;
```

---

## 4. Essential Analytical Functions and Abstraction

- **Time-series analysis:** Use `LAG()` and `LEAD()` to compare adjacent rows,
  avoiding expensive self-joins.
- **Ranking data:** Use `RANK()` for gap-aware ranking or `DENSE_RANK()` for tight
  rank numbering.
- **Abstracting windows:** Define a reusable window once with the `WINDOW`
  clause and reference it across multiple expressions.

Example:

```sql
SELECT
  year,
  production,
  min(production) OVER w AS min_prod,
  max(production) OVER w AS max_prod,
  avg(production) OVER w AS avg_prod
FROM oil_output
WINDOW w AS (ORDER BY year);
```

---

## Example prompt

- "Use PostgreSQL window functions to calculate a 3-point moving average and
  running total for monthly revenue."
- "Apply `LAG()` and `RANK()` to compare this year’s sales to prior periods and
  rank regions by performance."
