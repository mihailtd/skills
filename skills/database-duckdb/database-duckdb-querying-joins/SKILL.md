---
name: database-duckdb-querying-joins

description: Guides the agent through filtering rows with WHERE (including date/expression comparisons), the join types DuckDB supports (left/right/inner/full/cross) and what each returns for unmatched rows, chaining joins across three or more tables, and why an unordered result needs an explicit ORDER BY.
---

# DuckDB Querying and Joins

Building on the schema from **database-duckdb-ddl-crud**, this skill covers
filtering and combining rows across tables — the part of SQL that does the actual
analytical work once data is loaded and structured.

## 1. Filtering with WHERE

`WHERE` takes ordinary boolean expressions, including function calls and date
comparisons:

```sql
-- authors born more than 100 years before today
SELECT *
FROM authors
WHERE (YEAR(CURRENT_DATE) - birth_year) > 100;

-- date columns compare directly against ISO-format string literals
SELECT *
FROM borrowers
WHERE member_since >= '2022-01-01';
```

## 2. Join Types

| Join | Returns |
|---|---|
| `INNER JOIN` | Only rows with a match on both sides — the default mental model for "combine these two tables." |
| `LEFT JOIN` | Every row from the left table, with `NULL`s filling in unmatched right-side columns. |
| `RIGHT JOIN` | The mirror of `LEFT JOIN` — every row from the right table, `NULL`s on the left where unmatched. |
| `FULL JOIN` | Every row from both tables — matched where possible, `NULL`s on whichever side didn't match. |
| `CROSS JOIN` | Every row of the left table paired with every row of the right table (the Cartesian product) — no join condition, and the result size is the product of both row counts. Reach for it deliberately (e.g. generating all date × category combinations for a report), not by accident from a missing `ON`/`WHERE` clause. |

```sql
-- LEFT JOIN: every book, author NULL if somehow missing
SELECT b.book_id, b.title, a.name
FROM books b
LEFT JOIN authors a ON b.author_id = a.author_id;

-- same LEFT JOIN with the tables reversed: every author, even ones with no books
SELECT a.name, b.book_id, b.title
FROM authors a
LEFT JOIN books b ON a.author_id = b.author_id;
```

Which table is "left" is about which side of the query you write first — swapping
`FROM`/`JOIN` order changes which side's unmatched rows survive as `NULL`-filled
rows, not just cosmetic ordering.

## 3. Multi-Table Joins

Chain joins to pull related data across more than two tables — each `JOIN` adds one
more table to the same query, filtered and projected together in a single pass:

```sql
SELECT bw.name AS borrower_name, b.title AS book_title, a.name AS author_name
FROM borrowings br
INNER JOIN books b ON br.book_id = b.book_id
INNER JOIN borrowers bw ON br.borrower_id = bw.borrower_id
INNER JOIN authors a ON b.author_id = a.author_id;
```

Filtering down to a single entity works the same way, just with an added `WHERE`:

```sql
SELECT b.title AS book_title
FROM books b
INNER JOIN borrowings br ON b.book_id = br.book_id
INNER JOIN borrowers bw ON br.borrower_id = bw.borrower_id
WHERE bw.name = 'John Smith';
```

## 4. Deterministic Output with ORDER BY

SQL does not guarantee row order unless the query specifies one — a result that
happens to come back sorted today can come back in a different order after a
schema change, a data reload, or a query-planner change, since nothing promised
that order in the first place. Any query whose result order matters (a report, a
test assertion, a UI list) needs an explicit `ORDER BY`:

```sql
SELECT bw.name AS borrower_name, b.title AS book_title
FROM borrowings br
INNER JOIN books b ON br.book_id = b.book_id
INNER JOIN borrowers bw ON br.borrower_id = bw.borrower_id
ORDER BY bw.name, b.title;
```

## Related guidance

- **database-duckdb-ddl-crud** — the table definitions and constraints these queries run against.
- **database-duckdb-cli** — running these statements interactively.
- **database-duckdb-aggregation** — `GROUP BY` and aggregate functions, the natural next step once rows are filtered and joined.
- **database-duckdb-views-and-subqueries** — date-arithmetic analytics, saved views, and tie-safe top-N queries built on top of these joins.
- **database-duckdb-getting-started** — the same joins/filtering from a Python-client angle, one layer below this skill's raw-SQL depth.
- **database-duckdb-analytics-patterns** — section 5 corrects the old-style comma-join anti-pattern toward the explicit `JOIN` syntax taught here.
