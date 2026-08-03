---
name: sql-antipatterns-ambiguous-groups

description: Detects the Ambiguous Groups antipattern — selecting a column that isn't in GROUP BY and isn't wrapped in an aggregate, expecting the database to "figure out" it should come from the row that produced MAX()/MIN() — which either errors or (MySQL/SQLite) silently returns an arbitrary row's value, and guides toward window functions, correlated subqueries, derived tables, or outer joins that make the "row with the extreme value per group" query unambiguous.
---

# SQL Antipatterns — Ambiguous Groups

This skill helps developers recognize when a `GROUP BY` query violates the
Single-Value Rule — selecting a column that isn't grouped and isn't
aggregated, hoping the database infers "the value from the row that had the
`MAX()`" — and guides them to a query form that says that explicitly.

---

## 1. Recognize the Antipattern

- **The tell:** a `GROUP BY` query's select-list includes a column that's
  neither named in `GROUP BY` nor wrapped in an aggregate function, on the
  assumption the database will pick "the right" value to go with an
  aggregate like `MAX()`:

  ```sql
  SELECT product_id, MAX(date_reported) AS latest, bug_id
  FROM Bugs JOIN BugsProducts USING (bug_id)
  GROUP BY product_id;
  ```

  The intent ("give me the `bug_id` of the bug with the latest date, per
  product") is reasonable — but nothing in the query actually says that.
- Most databases reject this outright at parse time (DB2, SQL Server,
  Oracle, PostgreSQL, and MySQL 5.7+ by default all error). If your
  database silently accepts it and returns something, see §3 — that's not
  the query working, that's a compatibility mode masking the bug.

---

## 2. Why It Breaks Down: the Single-Value Rule

Every column in a `GROUP BY` query's select-list must be guaranteed to have
exactly one value per group. Columns named in `GROUP BY` satisfy this by
definition (that's what defines the group), and aggregate functions
(`MAX()`, `SUM()`, `COUNT()`, ...) satisfy it by computing one value across
the group. Any other column doesn't — the database has no principled way
to pick which of the group's many rows that column's value should come
from.

This isn't a database being overly strict — the "obviously correct" value
often doesn't exist:

- **Ties.** If two bugs share the exact same latest `date_reported`, which
  `bug_id` should the query return? Both are equally "the latest."
- **Multiple aggregates pointing at different rows.** `MAX(date_reported)`
  and `MIN(date_reported)` almost certainly come from two different rows
  in the group — there's no single `bug_id` that's simultaneously "the
  one with the max" and "the one with the min."
- **Aggregates with no corresponding row at all.** `SUM()`, `AVG()`, and
  `COUNT()` compute a value that may not match *any* individual row's
  value — there's no row to pull an extra column's value from in the
  first place.

A database that let this "sometimes work" (only erroring when the data
happened to be ambiguous) would be worse, not better — the same query
text would be valid or invalid depending on what's currently in the table,
which is not something you can catch in code review or a type check.

---

## 3. Legitimate Uses

- **Functional dependency.** If the select-list column is guaranteed
  unique per group value by the schema itself — most commonly, grouping
  by a table's primary key (or a foreign key referencing another table's
  primary key) and selecting that other table's attributes — there's
  genuinely only one possible value per group, even though the column
  isn't literally in `GROUP BY`:

  ```sql
  SELECT b.reported_by, a.account_name
  FROM Bugs b JOIN Accounts a ON (b.reported_by = a.account_id)
  GROUP BY b.reported_by;
  ```

  `account_name` is functionally dependent on `reported_by` because
  `reported_by` references `Accounts`' primary key. MySQL and SQLite allow
  this without complaint; most other brands still reject it (per the SQL
  standard) even though it's logically sound, because detecting functional
  dependency in general is hard for the optimizer to verify. If you rely
  on this in MySQL/SQLite, restrict yourself to genuinely
  functionally-dependent columns — not just ones that happen to be unique
  in today's data.

  **PostgreSQL (9.1+) is a special case worth calling out separately from
  MySQL/SQLite**, because it implements this exception as a deliberate,
  standards-compliant optimizer feature rather than a loose compatibility
  mode: if the `GROUP BY` list includes every column of a table's
  `PRIMARY KEY` (or a `UNIQUE`/`NOT NULL` constraint the planner can prove
  is a key), PostgreSQL allows any other column *from that same table* in
  the select-list unaggregated — because the primary key already
  guarantees at most one row per group, so every other column of that row
  is provably single-valued too. The same principle then transitively
  covers a joined table's columns whenever the join is on that table's
  primary key, exactly like the `Accounts` example above:

  ```sql
  -- valid on PostgreSQL: date_reported and status are functionally
  -- dependent on bug_id, which is Bugs' primary key
  SELECT bug_id, date_reported, status
  FROM Bugs
  GROUP BY bug_id;
  ```

  This is a real difference from MySQL/SQLite's behavior: Postgres proves
  the dependency from the schema's declared constraints (so it's sound —
  it can't accidentally paper over a genuinely ambiguous query), rather
  than skipping the check and returning whatever row physical storage
  order happens to produce. Don't over-extend it, though — it only
  recognizes dependency through a *declared* primary key/unique
  constraint, not through data that merely happens to be unique today, and
  not through more general functional dependencies (e.g. a `UNIQUE`
  composite of two non-key columns) unless they're declared as a real key
  constraint.
- MySQL (pre-5.7 default, or with `ONLY_FULL_GROUP_BY` disabled) and
  SQLite don't error on a truly ambiguous column — they silently return a
  value from an arbitrary row (MySQL: first row in physical storage order;
  SQLite: the last). This is undocumented, unstable across versions, and
  not something to rely on deliberately — treat it as a bug waiting to
  surface, not a feature.
- `GROUP BY` with no aggregate functions at all is equivalent to
  `SELECT DISTINCT` over the same columns — that's a legitimate,
  unambiguous use, just prefer `DISTINCT` for clarity when there's no
  aggregation happening.

---

## 4. The Fix: Make the "Row per Group" Query Unambiguous

Pick based on what your database supports and what the query needs to
scale to:

### a. Drop the ambiguous column

If you don't actually need `bug_id`, just don't select it — the simplest
fix is often overlooked:

```sql
SELECT product_id, MAX(date_reported) AS latest
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

### b. Window function (preferred where supported)

`ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` numbers rows within
each group by your chosen order, so filtering `rownum = 1` gets exactly
"the row that would have produced `MAX()`/`MIN()`," with all its other
columns intact:

```sql
SELECT t.product_id, t.date_reported, t.bug_id
FROM (
    SELECT bp.product_id, b.date_reported, b.bug_id,
           ROW_NUMBER() OVER (PARTITION BY bp.product_id
                               ORDER BY b.date_reported DESC) AS rownum
    FROM Bugs b JOIN BugsProducts bp USING (bug_id)
) AS t
WHERE t.rownum = 1;
```

Requires a reasonably modern database version (e.g. MySQL 8.0+) — check
window function support before relying on this.

### c. Correlated subquery

Readable, but re-executes the subquery once per outer row — weigh against
performance for large tables:

```sql
SELECT bp1.product_id, b1.date_reported AS latest, b1.bug_id
FROM Bugs b1 JOIN BugsProducts bp1 USING (bug_id)
WHERE NOT EXISTS (
    SELECT * FROM Bugs b2 JOIN BugsProducts bp2 USING (bug_id)
    WHERE bp1.product_id = bp2.product_id
      AND b1.date_reported < b2.date_reported
);
```

### d. Derived table

Compute the per-group extreme value once as a subquery, then join back to
pull the full row. Non-correlated, so it typically executes once — but
still materializes an intermediate result:

```sql
SELECT m.product_id, m.latest, b1.bug_id
FROM Bugs b1 JOIN BugsProducts bp1 USING (bug_id)
JOIN (SELECT bp2.product_id, MAX(b2.date_reported) AS latest
      FROM Bugs b2 JOIN BugsProducts bp2 USING (bug_id)
      GROUP BY bp2.product_id) m
  ON (bp1.product_id = m.product_id AND b1.date_reported = m.latest);
```

This can return more than one row per group if the max value ties across
multiple rows — wrap with an outer `GROUP BY`/`MAX(bug_id)` if a single row
per group is required.

### e. Outer join against a "nothing greater exists" condition

Scales well but is the hardest to read at a glance — worth the investment
when performance on large data matters more than readability:

```sql
SELECT bp1.product_id, b1.date_reported AS latest, b1.bug_id
FROM Bugs b1 JOIN BugsProducts bp1 ON (b1.bug_id = bp1.bug_id)
LEFT OUTER JOIN (Bugs b2 JOIN BugsProducts bp2 ON (b2.bug_id = bp2.bug_id))
  ON (bp1.product_id = bp2.product_id
      AND (b1.date_reported < b2.date_reported
           OR (b1.date_reported = b2.date_reported AND b1.bug_id < b2.bug_id)))
WHERE b2.bug_id IS NULL;
```

### f. Wrap the extra column in another aggregate

Only correct if you can independently guarantee the aggregates line up
(e.g. rows are always inserted in date order, so `MAX(bug_id)` really does
correspond to `MAX(date_reported)`) — verify that assumption before
relying on it, since nothing enforces it going forward:

```sql
SELECT product_id, MAX(date_reported) AS latest, MAX(bug_id) AS latest_bug_id
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

### g. Concatenate all group values instead of picking one

When you actually want every value in the group, not just the one tied to
the extreme — `GROUP_CONCAT()` (MySQL/SQLite) or an equivalent custom
aggregate (PostgreSQL) — but note this doesn't tell you *which* concatenated
value corresponds to the extreme, and isn't standard SQL:

```sql
SELECT product_id, MAX(date_reported) AS latest, GROUP_CONCAT(bug_id) AS bug_id_list
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

### h. Proprietary "I don't care which" escape hatches

MySQL's `ANY_VALUE()` explicitly suppresses the Single-Value Rule check
and returns an arbitrary row's value from the group — appropriate only
when you've independently verified the column is functionally dependent
(and the optimizer just can't prove it), or you genuinely don't care which
value comes back. Relying on "don't care" arbitrary behavior long-term is
fragile — prefer one of the deterministic options above when the answer
actually matters.

---

## 5. Review Checklist

- Does any `GROUP BY` query select a column that's neither grouped nor
  aggregated, relying on the database to "guess" which row it should come
  from?
- If it currently "works" on MySQL/SQLite, is that because of a genuine
  functional dependency (grouping by a primary/foreign key), or because
  `ONLY_FULL_GROUP_BY` happens to be off and an arbitrary row's value is
  being returned undetected?
- For "the row with the max/min value per group" queries, is the chosen
  technique (window function, correlated subquery, derived table, outer
  join) appropriate for the table size and the database's feature support?
- If an aggregate function is being used on an "extra" column just to
  satisfy the rule (`MAX(bug_id)`), is the assumption that makes it
  correct (e.g. insertion order) actually guaranteed, or just currently
  true?
