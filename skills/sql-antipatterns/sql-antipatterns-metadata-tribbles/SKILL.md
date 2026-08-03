---
name: sql-antipatterns-metadata-tribbles

description: Detects the Metadata Tribbles antipattern — cloning tables or columns per distinct data value (Bugs_2021, Bugs_2022, or bugs_fixed_2021, bugs_fixed_2022) to keep individual tables small — which forces new data to require new schema objects, and guides toward horizontal/vertical partitioning or a proper dependent table instead.
---

# SQL Antipatterns — Metadata Tribbles

This skill helps developers recognize when tables or columns are being
cloned per distinct data value (usually per year) to keep individual tables
"small," and guides them toward mechanisms that get the same performance
benefit without making new data require new schema.

---

## 1. Recognize the Antipattern

- **The tell, table form:** a new table is created for each value of some
  attribute, usually a year, with the value baked into the table name:

  ```sql
  CREATE TABLE Bugs_2019 ( ... );
  CREATE TABLE Bugs_2020 ( ... );
  CREATE TABLE Bugs_2021 ( ... );
  ```

- **The tell, column form:** the same thing one level down — a new column
  per distinct value instead of a new table:

  ```sql
  ALTER TABLE Customers ADD (revenue2002 NUMBER(9,2));
  ALTER TABLE Customers ADD (revenue2003 NUMBER(9,2));
  ALTER TABLE Customers ADD (revenue2004 NUMBER(9,2));
  ```

- **This is the mirror image of [[sql-antipatterns-entity-attribute-value]]
  and [[sql-antipatterns-polymorphic-associations]].** Those antipatterns
  turn metadata (a column or table name) into a data value stored in a
  string column. Metadata Tribbles does the opposite: it turns a *data
  value* (a year, a category) into metadata (a table or column name).
  Both directions cause trouble — mixing data and metadata is the common
  thread.
- **Listen for these phrases**:
  - "Then we need to create a table (or column) per ..." — splitting by
    distinct values in a column is exactly this antipattern taking shape.
  - "What's the maximum number of tables (or columns) the database
    supports?" — if you're worried about hitting that ceiling, the design
    is already wrong; a sensible schema never gets close to it.
  - "We forgot to create the new table for the new year, so inserts failed
    this morning." — the direct, recurring consequence: new *data*
    requiring new *schema* means an operational failure mode every time
    that boundary is crossed (new year, new category, ...) and someone
    forgets the corresponding migration.
  - "How do I query many tables at once? They all have the same columns."
    — a sign the tables should have been one table with an extra
    discriminator column all along.
  - "How do I pass a table name as a parameter, built from a year number?"
    — dynamic SQL to work around a design that shouldn't require it.

---

## 2. Why It Breaks Down

- **New data literally cannot be inserted until schema catches up.**
  Crossing into a new year with no `Bugs_2023` table yet means every
  insert for that year fails until someone manually creates it — a
  scheduling/ops dependency the application shouldn't have.
- **Data integrity has no automatic enforcement.** Nothing stops a 2022
  bug from landing in `Bugs_2021` by mistake; the only fix is a `CHECK`
  constraint per table that itself must be kept in sync (and updated
  every year) by hand.
- **Correcting a value can require moving the row to a different table.**
  If an attribute used to split the tables changes (e.g. a bug's reported
  date is corrected across a year boundary), a simple `UPDATE` isn't
  enough — the row now belongs in a different table and must be
  `INSERT`ed there and `DELETE`d from the old one.
- **Uniqueness across split tables needs extra machinery.** Primary keys
  generated per-table can collide across tables; guaranteeing global
  uniqueness needs a shared sequence, or an extra ID-generator table on
  databases without native sequences.
- **Cross-table queries mean `UNION`ing every table**, and that `UNION`
  has to be extended by hand every time a new table is added:

  ```sql
  SELECT b.status, COUNT(*) FROM (
      SELECT * FROM Bugs_2020
      UNION SELECT * FROM Bugs_2021
      UNION SELECT * FROM Bugs_2022
  ) AS b GROUP BY b.status;
  ```

- **Schema changes multiply.** Adding one column (e.g. `hours`) means
  altering every split table individually, and any `UNION` query across
  them breaks unless every table has matching columns.
- **Referential integrity from or to a split table is broken.** A foreign
  key must reference exactly one table, so a dependent table like
  `Comments` can't declare `FOREIGN KEY (bug_id) REFERENCES Bugs_????`.
  Querying "all bugs reported by this person regardless of year" needs a
  join against a `UNION` of every split table instead of one clean join.

---

## 3. Legitimate Uses

- **Archiving.** If historical data genuinely doesn't need to be queried
  alongside current data, it's reasonable to move older rows to a
  separate, compatible table (or database) to keep the active table lean
  — the key difference from the antipattern is that this split is a
  deliberate, occasional operational action, not the routine mechanism for
  handling new data.
- **Deliberate database-per-tenant sharding at genuine scale.** Splitting
  data into separate databases per customer/tenant (as WordPress.com does
  across thousands of blogs) is a legitimate operational strategy once a
  single database becomes the bottleneck for backup, restore, or load
  balancing — very different from splitting by an incidental attribute
  like a year. The distinguishing feature: the split boundary is a stable
  operational unit (a tenant), not an ever-growing sequence (a calendar
  year) that guarantees new schema objects on a fixed cadence.

---

## 4. The Fix: Partition and Normalize

Get the performance benefit of smaller physical tables without making new
data require new schema.

### a. Horizontal partitioning (sharding)

Let the database split rows across physical partitions internally while
still presenting one logical table to every query:

```sql
CREATE TABLE Bugs (
    bug_id         SERIAL PRIMARY KEY,
    date_reported  DATE
) PARTITION BY HASH (bug_id)
PARTITIONS 4;
```

Rows are never misplaced (the database enforces the partitioning rule, not
application code), and ordinary queries against `Bugs` need no knowledge
of the partition structure at all. The partition count doesn't need to track
a growing value like "current year" — a fixed number of partitions simply
holds more rows per partition as data grows, until you deliberately decide
to repartition. Syntax and specifics vary by database brand (this isn't
standardized in SQL), but every major engine supports some form of it.

**PostgreSQL note:** declarative partitioning (`PARTITION BY RANGE`/`LIST`/
`HASH`, native since PG10, meaningfully matured through PG14–18 — better
partition pruning, `MERGE`, foreign keys pointing *at* partitioned tables,
partitioned indexes created automatically per-partition) is exactly the
built-in tool that removes the excuse for hand-rolling `Bugs_2021`,
`Bugs_2022`, ... tables in the first place. The "split by year" case from
this chapter maps directly onto range partitioning, and — unlike the
manual approach — the database enforces which partition a row belongs in,
so misfiled rows (§2's "2022 bugs ended up in `Bugs_2021`" scenario) are
structurally impossible instead of a recurring data-integrity chore:

```sql
CREATE TABLE Bugs (
    bug_id        BIGSERIAL,
    date_reported DATE NOT NULL,
    -- ...
    PRIMARY KEY (bug_id, date_reported)
) PARTITION BY RANGE (date_reported);

CREATE TABLE Bugs_2024 PARTITION OF Bugs
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE TABLE Bugs_2025 PARTITION OF Bugs
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- queries never reference partitions directly:
SELECT * FROM Bugs WHERE date_reported >= '2024-06-01';  -- planner prunes to Bugs_2024 automatically
```

This still requires creating each new partition ahead of the boundary it
covers (Postgres doesn't auto-create yearly partitions for you — that's
typically handled with a scheduled job, or `pg_partman` for automated
partition management), so the "did we remember to create next year's
partition" operational risk from §2 isn't eliminated, just made much
cheaper and safer to get right (a partition-create statement, not a whole
new table plus every query that unions across the family). The
`CHECK`-constraint-based inheritance partitioning from older Postgres
versions (pre-10) has the same manual-per-table drawbacks as the
antipattern itself — prefer declarative partitioning on any currently
supported version.

### b. Vertical partitioning

Split by columns instead of rows when some columns are bulky or rarely
needed together with the rest of the row — large `BLOB`/`TEXT` columns are
the classic case:

```sql
CREATE TABLE ProductInstallers (
    product_id      BIGINT UNSIGNED PRIMARY KEY,
    installer_image BLOB,
    FOREIGN KEY (product_id) REFERENCES Products(product_id)
);
```

Queries against the main table that don't need the bulky column run
faster because they never touch it — but only if the query avoids `SELECT
*`, which would defeat the point by pulling the separated column back in
anyway.

### c. A proper dependent table for Metadata Tribbles columns

Same remedy as [[sql-antipatterns-multicolumn-attributes]]: replace
per-value columns with one narrow table and one row per value:

```sql
CREATE TABLE ProjectHistory (
    project_id BIGINT,
    year       SMALLINT,
    bugs_fixed INT,
    PRIMARY KEY (project_id, year),
    FOREIGN KEY (project_id) REFERENCES Projects(project_id)
);
```

New years are new *rows*, not new columns — no `ALTER TABLE`, and no
query needs editing when the calendar advances.

---

## 5. Review Checklist

- Does any table or column name embed a data value (a year, a category)
  that's expected to keep incrementing?
- Would onboarding a new value (a new year, a new region) currently
  require a schema migration before the application can accept data for
  it?
- Are there `UNION` queries stitched together across a family of
  identically-shaped tables, that need to be edited whenever a new table
  in that family is added?
- Is there a `CHECK` constraint (or worse, nothing at all) standing in for
  what should be automatic, database-enforced partitioning?
- Can a dependent/child table declare a real `FOREIGN KEY` to the parent,
  or does the parent being split make that impossible?
- If rows are split for performance, would the database's own horizontal
  or vertical partitioning features get the same benefit without forcing
  new data to require new schema objects?

---

## Related guidance

PostgreSQL-specific remedy:

- database-postgresql-declarative-partitioning
