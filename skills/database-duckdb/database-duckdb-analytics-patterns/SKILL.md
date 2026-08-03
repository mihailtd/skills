---
name: database-duckdb-analytics-patterns

description: Guides the agent through CTEs (WITH) for multi-stage aggregation, CASE WHEN bucketing (and DuckDB's GROUP BY ALL shortcut for it), the SUM(CASE WHEN ...)/COUNT(*) percentage-of-group idiom, data-quality checks with SIMILAR TO pattern matching, and why an implicit comma-join should be rewritten as an explicit JOIN.
---

# DuckDB Descriptive-Analytics SQL Patterns

Beyond single-table `GROUP BY` (**database-duckdb-aggregation**), real descriptive
analytics — "what percentage of X had property Y, broken down by Z" — tends to
reuse a small set of SQL shapes. This skill collects them.

## 1. CTEs for Multi-Stage Aggregation

`WITH name AS (...)` names a subquery so later parts of the same statement can
reference it — useful when a result depends on two different levels of
aggregation over the same table (e.g. "each group's count" *and* "each group's
share of a larger total" in the same row):

```sql
WITH per_day AS (
    SELECT origin_airport, destination_airport, day_of_week, COUNT(*) AS flights_that_day
    FROM flights
    WHERE cancelled = 0
    GROUP BY origin_airport, destination_airport, day_of_week
),
per_route AS (
    SELECT origin_airport, destination_airport, COUNT(*) AS flights_total
    FROM flights
    WHERE cancelled = 0
    GROUP BY origin_airport, destination_airport
)
SELECT
    per_day.origin_airport, per_day.destination_airport, per_day.day_of_week,
    100.0 * per_day.flights_that_day / per_route.flights_total AS pct_of_route_flights
FROM per_day
JOIN per_route
    ON per_day.origin_airport = per_route.origin_airport
    AND per_day.destination_airport = per_route.destination_airport;
```

Each CTE is just a named, reusable subquery — nothing DuckDB-specific about the
syntax itself, but it's the cleanest way to express "compute this aggregate at two
different granularities, then combine them," which a single `GROUP BY` can't do
in one pass.

## 2. CASE WHEN Bucketing — and `GROUP BY ALL`

`CASE WHEN ... END` turns a continuous or free-form value into a named bucket,
usable anywhere an expression is — including `GROUP BY`:

```sql
SELECT
    CASE
        WHEN scheduled_departure BETWEEN '0000' AND '0559' THEN '00:00-06:00'
        WHEN scheduled_departure BETWEEN '0600' AND '1159' THEN '06:00-12:00'
        WHEN scheduled_departure BETWEEN '1200' AND '1759' THEN '12:00-18:00'
        ELSE '18:00-24:00'
    END AS departure_slot,
    AVG(arrival_delay) AS avg_arrival_delay
FROM flights
WHERE arrival_delay > 0
GROUP BY ALL
ORDER BY departure_slot;
```

Standard SQL requires repeating the *entire* `CASE` expression in both `SELECT`
and `GROUP BY` verbatim (they don't share by reference) — tolerable once, brittle
the moment the bucketing logic changes in one place and not the other. DuckDB's
`GROUP BY ALL` is the fix: it groups by every non-aggregated column in the
`SELECT` list automatically, so the bucketing logic lives in exactly one place.
Prefer `GROUP BY ALL` over a repeated expression whenever the grouping key is
itself a nontrivial expression, not just a bare column.

## 3. Percentage-of-Group with SUM(CASE WHEN ...)

Counting how many rows in a group satisfy a condition, as a percentage of the
group, is `SUM` over a `CASE`-produced 0/1 flag divided by `COUNT(*)`:

```sql
SELECT
    airline,
    100.0 * SUM(CASE WHEN cancelled = 1 THEN 1 ELSE 0 END) / COUNT(*) AS cancelled_pct
FROM flights
GROUP BY airline
ORDER BY cancelled_pct DESC;
```

`SUM(cancelled) * 100.0 / COUNT(*)` works identically when the flag column is
already `0`/`1` (as `cancelled` is here) — the explicit `CASE WHEN` form is only
necessary when the condition isn't already a boolean/0-1 column. Either way,
multiply by `100.0` (not `100`) — integer division truncates, and this is the
easiest place to silently lose the fractional part of a percentage.

## 4. Data-Quality Checks with Pattern Matching

Before trusting a column's values in an analysis, verify what's actually in it —
`SIMILAR TO` (or `NOT SIMILAR TO`) with a regex-style pattern is a quick way to
find values that don't match an assumed shape:

```sql
-- rows where the airport code isn't purely alphabetic, contrary to assumption
SELECT year, month, day, origin_airport, destination_airport
FROM flights
WHERE origin_airport NOT SIMILAR TO '[A-Za-z]+'
   OR destination_airport NOT SIMILAR TO '[A-Za-z]+';
```

Treat a nonempty result here as a signal to understand *why* before proceeding —
in a real dataset, numeric-looking codes in an "airport code" column are often a
legitimate but different case (a system-assigned code, a diversion, a data-entry
artifact) that changes how later aggregations should handle those rows, not
something to filter out reflexively.

## 5. Prefer Explicit JOIN Over Comma-Style Implicit Joins

`FROM a, b WHERE a.key = b.key` and `FROM a JOIN b ON a.key = b.key` are
equivalent for an inner join, but the comma form puts the join condition in
`WHERE`, indistinguishable at a glance from an actual filter — and it's easy to
forget the condition entirely and silently get a full cross join instead. Always
write the explicit form (see **database-duckdb-querying-joins**), even for a
simple two-table inner join:

```sql
-- avoid
SELECT COUNT(a.airline), a.airline FROM flights f, airlines a
WHERE a.iata_code = f.airline AND f.arrival_delay > 0
GROUP BY a.airline;

-- prefer
SELECT COUNT(a.airline), a.airline
FROM flights f
JOIN airlines a ON a.iata_code = f.airline
WHERE f.arrival_delay > 0
GROUP BY a.airline;
```

## Related guidance

- **database-duckdb-aggregation** — the single-table `GROUP BY` this skill's multi-stage/CTE patterns build on.
- **database-duckdb-views-and-subqueries** — `CREATE VIEW` for persisting a query like the ones here, and the tie-safe top-N pattern.
- **database-duckdb-querying-joins** — the explicit join syntax section 5 corrects the comma-join anti-pattern toward.
- **database-duckdb-spatial-extension** — the geospatial half of this chapter's analysis, alongside these descriptive-analytics patterns.
- **database-duckdb-json-io** — `unnest()` paired with a CTE/subquery, the pattern section 1 here generalizes.
