---
name: database-duckdb-views-and-subqueries

description: Guides the agent through date-arithmetic analytics with DATEDIFF, saving a reusable query as a CREATE VIEW, and the nested-subquery-plus-HAVING pattern for finding every row tied for a maximum (rather than silently picking one, the pitfall in database-duckdb-aggregation's ORDER BY + LIMIT idiom).
---

# DuckDB Views and Subqueries

This skill covers three patterns that go beyond a single `GROUP BY`: computing
derived values like "days overdue" from date columns, persisting a query so it's
reusable like a table, and correctly handling ties when "the top row" isn't
actually unique.

## 1. Date Arithmetic for Analytics

`DATEDIFF('day', start, end)` computes the difference between two dates — the
building block for anything shaped like "how many days late," "how long ago,"
or "days until."

```sql
-- books returned more than 14 days after borrowing, and by how many days
SELECT bw.name AS borrower_name, b.title AS book_title,
       br.borrow_date, br.return_date,
       DATEDIFF('day', br.borrow_date, br.return_date) - 14 AS overdue
FROM borrowings br
INNER JOIN books b ON br.book_id = b.book_id
INNER JOIN borrowers bw ON br.borrower_id = bw.borrower_id
WHERE br.return_date IS NOT NULL
  AND DATEDIFF('day', br.borrow_date, br.return_date) > 14;
```

`IS NOT NULL` on `return_date` is what scopes this to books that have actually
been returned — for books still on loan, use `IS NULL` there instead and compare
against `CURRENT_DATE()` (or a specific date, for "was this overdue as of a
particular point in time" rather than "as of right now") in place of
`return_date`:

```sql
-- currently on loan and already past the 14-day window
SELECT b.book_id, b.title, bw.name AS borrower_name, br.borrow_date,
       DATEDIFF('day', br.borrow_date, CURRENT_DATE()) - 14 AS overdue
FROM borrowings br
INNER JOIN books b ON br.book_id = b.book_id
INNER JOIN borrowers bw ON br.borrower_id = bw.borrower_id
WHERE br.return_date IS NULL
  AND DATEDIFF('day', br.borrow_date, CURRENT_DATE()) > 14;
```

Swapping `CURRENT_DATE()` for a literal date (`'2022-06-10'`) answers "what was
overdue as of that day" instead of "as of right now" — useful for reproducible
reports that shouldn't give a different answer depending on when they're rerun.

## 2. Saving a Query as a View

`CREATE VIEW` persists a query under a name you can `SELECT FROM` like a table —
the query itself isn't re-typed or re-copy-pasted everywhere it's needed, and it
re-runs against current data every time the view is queried (it's a saved query,
not a saved snapshot):

```sql
CREATE VIEW overdue_borrowings AS
SELECT bw.name AS borrower_name,
       b.title AS book_title,
       br.borrow_date,
       br.return_date,
       DATEDIFF('day', br.borrow_date, br.return_date) - 14 AS overdue
FROM borrowings br
INNER JOIN books b ON br.book_id = b.book_id
INNER JOIN borrowers bw ON br.borrower_id = bw.borrower_id
WHERE br.return_date IS NOT NULL
  AND DATEDIFF('day', br.borrow_date, br.return_date) > 14;

SELECT * FROM overdue_borrowings;
```

Reach for a view when a query is genuinely reused across multiple later queries or
reports in the same database — for a one-off analysis inside a single Python
session, a Polars/DuckDB variable (see **database-duckdb-getting-started**) does
the same job without adding a persistent database object to maintain.

## 3. Handling Ties Correctly with Nested Subqueries

**database-duckdb-aggregation** section 2 covers `ORDER BY ... DESC LIMIT 1` for
finding a single top row — and its blind spot: a tie gets silently resolved to one
arbitrary row. When every tied row genuinely matters (multiple books borrowed the
same, highest number of times), compute the maximum separately and match against
it with `HAVING`/`IN` instead of `LIMIT`:

```sql
SELECT b.book_id, b.title AS book_name,
       bw.name AS borrower_name,
       br.borrow_date AS loan_date, br.return_date
FROM borrowings br
INNER JOIN books b ON br.book_id = b.book_id
INNER JOIN borrowers bw ON br.borrower_id = bw.borrower_id
WHERE b.book_id IN (
    SELECT book_id
    FROM borrowings
    GROUP BY book_id
    HAVING COUNT(*) = (
        SELECT MAX(num_borrowings)
        FROM (
            SELECT COUNT(*) AS num_borrowings
            FROM borrowings
            GROUP BY book_id
        ) AS counts
    )
);
```

Read it from the inside out:

1. **Innermost** — count borrowings per `book_id`.
2. **Middle** — take the `MAX` of those per-book counts: the single highest
   borrow count, whatever it happens to be.
3. **`HAVING COUNT(*) = ...`** — keep every `book_id` whose own count matches
   that maximum, not just the first one encountered — this is what makes ties
   surface instead of disappear.
4. **Outer `SELECT`** — join back to `books`/`borrowers` to attach the
   human-readable details (title, borrower, dates) to the `book_id`s that
   survived the filter.

This is the same "compute the aggregate, then filter rows against it" shape as
**database-duckdb-aggregation** section 3's `MIN`-row lookup, just with `IN`/
`HAVING` instead of `=`/`WHERE` because more than one row can legitimately match.

## Related guidance

- **database-duckdb-aggregation** — `GROUP BY`, aggregate functions, and the `LIMIT`-based top-N pattern this skill's tie-safe version replaces when ties matter.
- **database-duckdb-querying-joins** — the join syntax used throughout these examples.
- **database-duckdb-getting-started** — the Python-client equivalent for a one-off query that doesn't need to be a persisted view.
- **database-duckdb-analytics-patterns** — CTEs (`WITH`) as a more general multi-stage-aggregation tool alongside subqueries and views.
