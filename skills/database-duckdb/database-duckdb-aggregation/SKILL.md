---
name: database-duckdb-aggregation

description: Guides the agent through GROUP BY and aggregate functions (COUNT/AVG/MIN/MAX) in DuckDB SQL, the ORDER BY + LIMIT idiom for finding a single top row, its silent-tie-breaking pitfall, and the scalar-subquery pattern for retrieving the full row behind an aggregate rather than just the aggregate value.
---

# DuckDB Aggregation

Building on the joins in **database-duckdb-querying-joins**, aggregation is where
SQL earns its keep for analytics — collapsing many rows into a summary value per
group. This skill covers the core patterns, plus two things that are easy to get
subtly wrong: silently dropping ties with `LIMIT`, and confusing "give me the
aggregate value" with "give me the whole row that produced it."

## 1. GROUP BY and Aggregate Functions

`GROUP BY` collapses rows sharing a column value into one row per group, and
aggregate functions (`COUNT`, `AVG`, `MIN`, `MAX`, `SUM`) summarize each group:

```sql
SELECT bw.name AS borrower_name, COUNT(br.book_id) AS books_borrowed
FROM borrowings br
INNER JOIN borrowers bw ON br.borrower_id = bw.borrower_id
GROUP BY bw.name
ORDER BY bw.name;

SELECT AVG(publication_year) AS avg_publication_year
FROM books;
```

Every selected column that isn't wrapped in an aggregate function must appear in
`GROUP BY` — SQL doesn't know which of a group's several values to show for a
column you didn't either aggregate or group by, so it's a hard requirement, not a
style preference. Aggregates also compose with date/expression logic, not just raw
columns:

```sql
SELECT AVG(YEAR(CURRENT_DATE) - birth_year) AS average_age_of_authors
FROM authors;
```

## 2. Top-N via ORDER BY + LIMIT — and Its Tie Pitfall

Sorting by the aggregate and taking the first row is the standard way to find a
single "winner":

```sql
SELECT b.book_id, b.title AS book_name, COUNT(*) AS num_borrowings
FROM borrowings br
INNER JOIN books b ON br.book_id = b.book_id
GROUP BY b.book_id, b.title
ORDER BY num_borrowings DESC
LIMIT 1;
```

This is correct when you genuinely want *one* row and don't care which one wins a
tie. It's the wrong tool when a tie is meaningful — if two books are borrowed the
same number of times, `LIMIT 1` silently picks whichever one the query planner
happens to return first, discarding the other without any indication a tie
occurred. See **database-duckdb-views-and-subqueries** section 3 for the pattern
that surfaces every tied row instead of picking one arbitrarily.

## 3. Finding the Row Behind an Aggregate

`MIN`/`MAX` alone gives you the *value*, not the row it came from. To get the full
row, filter with a scalar subquery that computes the aggregate separately:

```sql
SELECT name, birth_year, YEAR(CURRENT_DATE) - birth_year AS age
FROM authors
WHERE birth_year = (SELECT MIN(birth_year) FROM authors);
```

The subquery `(SELECT MIN(birth_year) FROM authors)` must return exactly one
value (a genuine scalar) for this to work as a plain `=` comparison — if it could
return multiple rows, use `IN` instead of `=`, the same distinction that shows up
again in the tie-safe pattern in **database-duckdb-views-and-subqueries**.

## Related guidance

- **database-duckdb-querying-joins** — the joins these aggregation queries build on top of.
- **database-duckdb-views-and-subqueries** — saving a query as a view, date-arithmetic analytics, and the tie-safe version of the top-N pattern from section 2.
- **database-duckdb-getting-started** — the same `GROUP BY`/`AVG`/`MAX` basics from the Python-client angle (`.pl()`), one layer below this skill's raw-SQL depth.
- **database-duckdb-analytics-patterns** — `GROUP BY ALL`, CTE-based multi-stage aggregation, and percentage-of-group calculations built on top of the basics here.
- **database-duckdb-spatial-extension** — this same `ORDER BY`/`LIMIT` top-N shape, applied to nearest-neighbor distance queries.
