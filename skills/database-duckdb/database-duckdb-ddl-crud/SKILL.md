---
name: database-duckdb-ddl-crud

description: Guides the agent through defining DuckDB tables with PRIMARY KEY/FOREIGN KEY constraints, inspecting schema (DESCRIBE, SHOW, .schema), dropping tables and the referential-integrity error that blocks it, and the CRUD statements (INSERT, UPDATE, DELETE) including the LIKE-wildcard delete and the danger of an unconditioned DELETE.
---

# DuckDB Table Definition and CRUD

DuckDB's DDL and CRUD statements follow standard SQL closely (DuckDB targets
SQL:1999-era compatibility) — if you already know SQL, this is mostly a
confirmation, not new syntax. The examples below build a small multi-table schema
(authors/books/borrowers/borrowings for a library) to show constraints and joins
in a setting that isn't a single flat table.

## 1. Defining Tables with Constraints

`PRIMARY KEY` and `NOT NULL` work as expected. `FOREIGN KEY ... REFERENCES` enforces
referential integrity — a value inserted into the referencing column must already
exist in the referenced table's key column:

```sql
CREATE TABLE authors (
    author_id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    nationality TEXT,
    birth_year INTEGER
);

CREATE TABLE books (
    book_id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    author_id INTEGER NOT NULL,
    genre TEXT,
    publication_year INTEGER,
    FOREIGN KEY (author_id) REFERENCES authors(author_id)
);
```

`FOREIGN KEY (author_id) REFERENCES authors(author_id)` means every `author_id`
inserted into `books` must already exist in `authors` — DuckDB rejects an insert
that references a nonexistent author rather than silently allowing an orphaned row.

## 2. Inspecting Schema

`DESCRIBE table_name` (or `SHOW table_name`) lists a table's columns, types,
nullability, and key role:

```sql
DESCRIBE authors;
```

`.schema` (a CLI dot command — see **database-duckdb-cli**) prints the full
`CREATE TABLE` statements for every table in the database in one shot, useful for
seeing the whole schema — including foreign keys — at a glance rather than one
table at a time.

## 3. Dropping Tables

```sql
DROP TABLE some_unused_table;
```

Dropping a table that's referenced by another table's `FOREIGN KEY` fails with a
catalog error rather than silently orphaning the referencing rows — this is the
constraint from section 1 doing its job. Drop the referencing table first, or
remove the constraint, if the drop is genuinely intended.

## 4. Inserting Rows

Multi-row `INSERT INTO ... VALUES` is the standard bulk-literal form:

```sql
INSERT INTO authors (author_id, name, nationality, birth_year)
VALUES
    (1, 'Jane Austen', 'British', 1775),
    (2, 'Charles Dickens', 'British', 1812),
    (3, 'Agatha Christie', 'British', 1890);
```

For values that come from outside the program rather than literals typed by hand,
use parameter binding, not string interpolation — see
**database-duckdb-getting-started** section 3 for why and how; the same rule
applies here regardless of whether the statement runs from the CLI or the Python
client.

## 5. Updating Rows

`UPDATE ... SET ... WHERE` — the `WHERE` clause is what scopes the update to
specific rows; without one, every row in the table gets the same new values:

```sql
UPDATE borrowings
SET return_date = '2022-04-05',
    status = 'Returned'
WHERE borrowing_id = 3;
```

## 6. Deleting Rows

`DELETE FROM ... WHERE` scopes a delete the same way `UPDATE` does:

```sql
DELETE FROM borrowers WHERE borrower_id = 6;
```

`LIKE` with `%` wildcards deletes by pattern instead of an exact match:

```sql
DELETE FROM borrowers WHERE name LIKE '%Jane%';
```

**Always double-check the `WHERE` clause before running a `DELETE` or `UPDATE`** —
a `DELETE FROM borrowers;` with no `WHERE` at all deletes every row in the table,
silently and without confirmation. This isn't a DuckDB-specific gotcha, but it's
worth stating explicitly: review the filter before executing, especially when a
statement is generated or assembled programmatically rather than typed by hand.

## Related guidance

- **database-duckdb-cli** — running these same statements interactively, plus `.schema`/`.tables` for inspection.
- **database-duckdb-getting-started** — the Python-client connection/parameter-binding mechanics these statements run through when called from a script.
- **database-duckdb-querying-joins** — `SELECT`, filtering, and joins across the tables defined here.
- **database-duckdb-json-io** — `DESCRIBE` used to verify inferred column types after loading JSON from multiple files.
