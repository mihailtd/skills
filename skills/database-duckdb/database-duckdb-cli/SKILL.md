---
name: database-duckdb-cli

description: Guides the agent through the DuckDB command-line shell — launching against an in-memory or persistent database, the most useful startup flags, dot commands for database administration (.tables, .schema, .open vs ATTACH/USE, .dump, .read), and persisting an in-memory session to disk with EXPORT DATABASE/IMPORT DATABASE — for tasks that don't need a Python process at all.
---

# The DuckDB CLI

Everything the Python client can do, the DuckDB CLI can do interactively from a
terminal — creating tables, importing data, running ad hoc SQL — without a Python
process in the loop. Reach for it for one-off database administration and
exploration; reach for the Python client (**database-duckdb-getting-started**) when
the result needs to feed back into a program.

## 1. Installing and Launching

Install via your platform's package manager — `brew install duckdb` on macOS,
`winget install DuckDB.cli` on Windows, or a native package/direct binary on Linux.

Launch with no filename for a transient in-memory database (destroyed on exit), or
with a filename for a persistent one (created on first use if it doesn't exist):

```bash
duckdb                  # in-memory, gone when the shell exits
duckdb mydb.duckdb       # persistent — reopens the same data next time
```

The prompt is `D`. Every SQL statement must end with `;` — omitting it just makes
the shell wait for more input on the next line, it doesn't run anything.

A few startup flags worth knowing beyond the defaults:

| Flag | Effect |
|---|---|
| `-readonly` | Open the database without write access — use for concurrent read access to a file another process has open (see **database-duckdb-getting-started** section 1 for the same tradeoff via the Python client's `read_only=True`) |
| `-c "SQL"` / `-s "SQL"` | Run one statement and exit — useful from a script or CI step instead of the interactive shell |
| `-csv` / `-json` / `-markdown` / `-table` | Set the output format for query results |
| `-no-stdin` | Exit after processing flags instead of dropping into the interactive prompt |

Run `duckdb -help` for the full flag list.

## 2. Loading Data Directly

The same bare-`FROM`/`read_csv_auto`/`read_parquet` functions from
**database-duckdb-csv-io** and **database-duckdb-parquet-io** work unchanged in the
CLI — there's no separate CLI-specific import syntax to learn:

```sql
D CREATE TABLE airlines AS FROM 'airlines.csv';
D SHOW TABLES;
D SELECT * FROM airlines;
```

## 3. Dot Commands

Commands prefixed with `.` are CLI-shell features, not SQL — they don't take a
trailing semicolon, and `.help` lists all of them. The ones worth knowing:

- **`.tables`** — list tables across all attached databases at a glance.
- **`.schema`** — print the `CREATE TABLE` statements for everything in the
  current database (or matching a pattern) — the fastest way to see a database's
  full structure in one shot.
- **`.database`** — show which database(s) are currently attached and which is
  active.
- **`.open FILENAME`** — close the current database and open a different one.
  This *replaces* the active connection. To keep the current database open
  *and* work with another one simultaneously, use `ATTACH` instead:
  ```sql
  D ATTACH 'other.duckdb' AS other;
  D USE other;   -- switch which attached database unqualified names resolve to
  ```
- **`.dump TABLE`** — render a table's contents as `INSERT` statements, useful
  for moving data into a different database engine or system. The table must be
  in the currently active database (`USE` into it first if it isn't).
- **`.read FILE`** — execute a `.sql` file's statements as if typed at the
  prompt — the CLI equivalent of running a script.
- **`.mode`, `.headers`, `.timer`** — control output formatting and whether
  query timing is printed; useful when scripting the CLI non-interactively.

## 4. Persisting an In-Memory Session

An in-memory session's data is gone the moment the shell exits. `EXPORT DATABASE`
writes every table out to a directory (one CSV per table, plus a `schema.sql` and a
`load.sql` describing how to reconstruct it) — a portable snapshot, not just a
DuckDB-specific dump:

```sql
D EXPORT DATABASE 'airports_db';
```

Rebuild it in a fresh (or different) database with `IMPORT DATABASE`:

```bash
$ duckdb newdb.duckdb
```
```sql
D IMPORT DATABASE 'airports_db';
D SHOW TABLES;
```

This is the CLI-native equivalent of `COPY`/`CREATE TABLE AS` for a whole database
at once, rather than one table at a time.

## Related guidance

- **database-duckdb-getting-started** — the Python-client equivalents (`connect`, `read_only`) for when the result needs to feed back into a program.
- **database-duckdb-ddl-crud** — table constraints, `DESCRIBE`, and CRUD statements, runnable identically from the CLI or the Python client.
- **database-duckdb-external-databases** — `ATTACH ... (TYPE POSTGRES)`, the same `ATTACH` mechanism used here for a second local `.duckdb` file, applied to a live PostgreSQL server instead.
