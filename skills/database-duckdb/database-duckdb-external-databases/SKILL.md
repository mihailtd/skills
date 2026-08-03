---
name: database-duckdb-external-databases

description: Guides the agent through pulling data from a live PostgreSQL server into DuckDB for analysis — DuckDB's postgres extension (INSTALL/LOAD/ATTACH, querying attached tables directly) versus the manual row-by-row driver loop — and why the extension is almost always the better default.
---

# DuckDB Against a Live Database Server (PostgreSQL)

Sometimes the data you want to analyze isn't a file — it's already sitting in a
running database server. DuckDB can pull from PostgreSQL (this repository's
relational database of choice — see the `python` master skill's database decision
guide) two ways: attach the server directly with the `postgres` extension, or
connect with a Python driver and copy rows over manually. Default to the
extension; reach for the manual path only when you specifically need to reshape
data mid-transfer.

## 1. The Manual Method — and Why to Avoid It for Volume

The manual approach is: open a driver connection, run a query, fetch rows, and
`INSERT` them into a DuckDB table one at a time in a loop. It works, and it gives
you full control to transform each row on the way through, but it pays a Python
round-trip and a separate `INSERT` statement per row — fine for a few hundred rows,
prohibitively slow for anything larger. This is the same anti-pattern flagged in
**database-duckdb-excel-io** section 3 for row-by-row Excel loads — a driver loop
belongs at Python-application scale (see the `python-postgresql` package's
SQLAlchemy-based repository pattern for that), not at data-analysis scale where
you're moving thousands or millions of rows for a one-off query.

If you do need the manual path — say, to filter or cast values while copying —
batch the inserts (`executemany`, or DuckDB's `appender` API) instead of one
`INSERT` per row.

## 2. The `postgres` Extension — Attach and Query Directly

Install once per DuckDB installation, same as any other extension:

```python
conn.execute("INSTALL postgres")
conn.execute("LOAD postgres")
```

`ATTACH` registers the live Postgres server as a queryable schema — no data copy,
no intermediate DuckDB table, the query runs against Postgres through DuckDB's SQL
layer:

```python
conn.execute("""
    ATTACH 'host=localhost port=5432 dbname=app_db user=app_user password=secret'
    AS pg (TYPE POSTGRES)
""")

conn.sql("SELECT * FROM pg.public.airlines").pl()
```

`USE pg` switches the default schema so unqualified table names resolve against
the attached database for the rest of the session, avoiding the `pg.public.`
prefix on every query:

```python
conn.execute("USE pg")
conn.sql("SELECT * FROM airlines").pl()
conn.sql("SHOW TABLES").pl()
```

Never interpolate a password into the `ATTACH` string via an f-string built from
user input — build the connection string from configuration/secrets the same way
this repository already sources database credentials for its SQLAlchemy
connections (see the `python` master skill's configuration guidance), not from
anything an end user can influence.

## 3. Decision Heuristic

- **Default to the extension** (section 2) — simpler, no per-row Python overhead,
  and the query planner can push filters down to Postgres itself.
- **Fall back to the manual method** (section 1) only when you need to transform,
  validate, or reshape rows during the transfer in ways SQL can't express cleanly,
  or in an environment where installing the extension isn't possible.
- **This is an analysis-time tool, not a replacement for your application's data
  layer** — the app still talks to PostgreSQL through SQLAlchemy
  (`python-postgresql` package); DuckDB's Postgres attachment is for pulling data
  out for ad hoc analysis, not for serving application requests.

## Related guidance

- **database-duckdb-overview** — the broader DuckDB-vs-application-database distinction.
- **database-duckdb-excel-io** — the same row-by-row-loop tradeoff, for spreadsheet sources instead of a database server.
- **database-duckdb-polars-postgresql-integration** — the deeper *why* behind this pattern (isolating analytical load, cross-source joins) versus the mechanics covered here.
- The `python` master skill (package `python-core`) — the PostgreSQL/MongoDB decision guide and SQLAlchemy repository pattern used for actual application persistence.
- **database-duckdb-marimo-notebooks** — using a connection like this one as a marimo notebook's custom DuckDB connection.
- **database-duckdb-huggingface-datasets** — another "attach a remote source" pattern (a hosted dataset repository instead of a database server), including its own credential-handling approach.
