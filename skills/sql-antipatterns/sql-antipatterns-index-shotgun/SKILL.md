---
name: sql-antipatterns-index-shotgun

description: Detects the Index Shotgun antipattern — adding or removing indexes by guesswork instead of by measurement (no indexes at all, indexes on every column/permutation, or indexes that no query can actually use) — and guides toward the MENTOR methodology (Measure, Explain, Nominate, Test, Optimize, Rebuild) for choosing indexes deliberately.
---

# SQL Antipatterns — Index Shotgun

This skill helps developers stop guessing at indexes — creating too few,
too many, or ones that no query can actually use — and instead choose
indexes based on measured query cost and execution plans.

---

## 1. Recognize the Antipattern

Guessing at indexes tends to produce one of three failure modes:

- **No indexes, or too few.** Reasoning like "indexes add overhead on
  every `INSERT`/`UPDATE`/`DELETE`, so removing them removes waste" ignores
  that the overhead buys back far more read performance than it costs,
  in any table read far more often than it's written — which is most
  tables. An index also speeds up `UPDATE`/`DELETE` statements that need to
  locate matching rows first.
- **Too many indexes, "just in case."** Indexes on columns no query
  actually filters, sorts, or joins on add storage and write overhead with
  no offsetting benefit. A redundant index on a primary key (most engines
  already index it automatically — check your brand's rules) is pure
  waste. A wide compound index whose column order doesn't match how
  queries actually use it is likely never chosen by the optimizer at all.
- **Indexes that no query can use.** An index only helps queries that
  filter/sort/join in a way compatible with its column order. This is the
  case developers most often misjudge — see §2.

**Listen for these phrases**:

- "Here's my query, how do I make it faster?" — asked with no table
  schema, index list, data volume, or measurement attached. Any answer at
  that point is a guess, not a diagnosis.
- "I put an index on every column, why isn't it faster?" — the antipattern
  by name: shooting at every possible target instead of aiming at a
  measured bottleneck.
- "I read that indexes make the database slow, so I don't use them." —
  a blanket rule standing in for actually looking at the workload.

---

## 2. Why Indexes Silently Fail to Help

An index is only usable in the way it's structured — like a phone book
sorted by last name, then first name: looking up a last name is fast,
looking up a first name still means scanning every entry. The same
mismatch happens in SQL:

- **Wrong leading column in a compound index.** `INDEX(last_name,
  first_name)` doesn't help `ORDER BY first_name, last_name` or `WHERE
  first_name = 'Charles'` — the index's sort order starts with last name,
  so first names are scattered throughout it.
- **Functions/expressions wrapping the indexed column.** `INDEX
  (date_reported)` can't help `WHERE MONTH(date_reported) = 4` — the index
  is ordered by full date, so April rows from every year are scattered
  through it, not adjacent. An index only helps the exact expression it
  was built on (some engines support expression/generated-column indexes
  for exactly this case, but only for the specific expression you index).
- **`OR` across differently-indexed columns.** `WHERE last_name =
  'Charles' OR first_name = 'Charles'` needs the equivalent of unioning two
  separate lookups — an index on `(last_name, first_name)` only accelerates
  the `last_name` half.
- **Leading wildcard / substring search.** `WHERE description LIKE
  '%crash%'` can't use a sorted index at all — the match could start
  anywhere in the string, so there's no ordering property to exploit —
  consider a full-text search feature instead of `LIKE '%...%'` for this
  case (see [[sql-antipatterns-poor-mans-search-engine]]).
- **Low selectivity.** Selectivity is `COUNT(DISTINCT col) / COUNT(col)` —
  the fraction of rows that share a value. An index on a column where most
  rows share a handful of values (e.g. a `status` with 3 possible values
  across a million rows) may cost more to consult than a full table scan,
  the same way a book index entry that lists half the book's pages isn't
  worth flipping through.
- **Indexes aren't standardized.** The SQL standard says nothing about
  indexes at all — syntax, automatic index creation rules (e.g. for
  primary keys), query plan reporting, and maintenance commands are all
  brand-specific. Don't assume behavior from one database transfers to
  another without checking that brand's documentation.

---

## 3. Legitimate Uses

- When designing a general-purpose schema with no specific query workload
  yet known, you have to make an educated guess at indexes — accept that
  you'll both miss some useful ones and add some that turn out unused,
  and plan to revisit once real query patterns exist (see the MENTOR loop
  below).
- Before blaming the database for slow performance at all, verify the
  database actually is the bottleneck — profile the application. It's
  common for the real cost to sit somewhere else entirely (parsing,
  rendering, network I/O), in which case no amount of indexing helps and
  the "database is always the bottleneck" assumption wastes the
  optimization effort on the wrong target.

---

## 4. The Fix: MENTOR Your Indexes

A deliberate loop beats guessing — **M**easure, **E**xplain, **N**ominate,
**T**est, **O**ptimize, **R**ebuild:

### Measure

Find out which queries actually cost the most, using your database's
logging/tracing tools (SQL Server Profiler, Oracle TKProf, MySQL's slow
query log, PostgreSQL's `log_min_duration_statement`). The most expensive
query overall isn't always the slowest one — a cheap query run constantly
can cost more in aggregate than a rare, slow one. Disable query result
caching while measuring, since it bypasses the index usage you're trying
to observe, and profile against real usage/data volume where possible.

### Explain

For the query you've identified, get the database's query execution plan
(`EXPLAIN` in MySQL/PostgreSQL/Oracle, `SET SHOWPLAN_XML` in SQL Server,
`db2expln`/Visual Explain in DB2). The plan's format is vendor-specific,
but in general it shows which indexes (if any) the optimizer chose, in
what order tables are accessed, and where it falls back to a full scan or
an unindexed sort.

### Nominate

From the plan, identify where the query touches a table without using an
index, and propose a specific index for that access pattern. Several
databases have automated advisors that do this analysis for you (DB2
Design Advisor, SQL Server Database Engine Tuning Advisor, MySQL
Enterprise Query Analyzer, Oracle Automatic SQL Tuning Advisor).

Consider a **covering index** — one that includes every column the query
needs, not just the ones used for filtering/sorting — so the query can be
satisfied from the index alone without reading the table rows at all:

```sql
CREATE INDEX BugCovering ON Bugs
    (status, bug_id, date_reported, reported_by, summary);

SELECT status, bug_id, date_reported, summary
FROM Bugs WHERE status = 'OPEN';  -- satisfied entirely from the index
```

### Test

Re-measure after adding the index. Confirm the change actually helped
before considering the work done — and keep the before/after numbers; "I
added one index and cut query time by 38%" is a concrete result, "I tried
some things and we'll see" is not.

### Optimize

Indexes are compact and heavily reused, which makes them ideal candidates
for staying in memory cache (an order of magnitude faster than disk I/O).
Default cache/buffer-pool sizes are usually tuned conservatively for
low-powered environments — increase them for a real production server
(e.g. MySQL's `innodb_buffer_pool_size`), watching disk I/O rates as you
tune rather than picking a number blindly.

### Rebuild

Indexes can become imbalanced over time as rows are inserted/updated/
deleted, similar to filesystem fragmentation. Schedule periodic
maintenance (`OPTIMIZE TABLE` in MySQL, `ALTER INDEX ... REBUILD` in SQL
Server/Oracle, `VACUUM`/`ANALYZE` in PostgreSQL/SQLite, `REORG INDEX` in
DB2) — but weigh the cost against the benefit for large, infrequently
updated, or rarely queried tables, where a full rebuild may not be worth
the time it takes.

---

## 5. Don't Index Every Column (or Every Permutation)

Trying to guarantee every query is covered by indexing "everything" is
worse than it sounds — a compound index's usefulness depends on column
*order*, so covering every possible query shape means indexing every
*permutation* of columns, not just every column:

```sql
CREATE TABLE Bugs (
    bug_id         SERIAL PRIMARY KEY,
    date_reported  DATE NOT NULL,
    summary        VARCHAR(80) NOT NULL,
    status         VARCHAR(10) NOT NULL,
    INDEX (bug_id, date_reported, summary, status),
    INDEX (date_reported, bug_id, summary, status),
    INDEX (summary, date_reported, bug_id, status),
    -- ... every permutation ...
);
```

The number of permutations is the factorial of the column count — 4
columns need 24 indexes, 5 columns need 120, and many databases won't
even allow that many indexes on one table. Every one of those indexes
adds storage and write overhead regardless of whether any query ever uses
it. Create only the indexes that support queries you actually run;
revisit when a new query pattern is added, using the MENTOR loop above
rather than pre-emptively indexing every shape a query could take.

---

## 6. Review Checklist

- Was each index added because a measured, slow query's execution plan
  showed it would help — or because "it seemed like it might"?
- Does any compound index's column order actually match how queries
  filter/sort/join, or is a differently-ordered index silently unused?
- Are there indexes on columns wrapped in functions (`MONTH(date)`,
  `LOWER(name)`) where the plain column index can't help the actual query?
- Are there indexes on low-selectivity columns (few distinct values across
  many rows) that likely cost more than the table scan they're meant to
  avoid?
- Is there a redundant index duplicating one the database already creates
  automatically (e.g. for a primary key)?
- Has anyone actually looked at an `EXPLAIN`/query execution plan for the
  slow query, or is the fix being guessed at directly?

---

## Related guidance

PostgreSQL-specific remedy:

- database-postgresql-unused-indexes
