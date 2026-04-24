---
name: python-polars-database-io
description: Instructs the agent on Polars database connectivity and query workflows, including URI formats, query pushdown, and using databases as external sources.
---

# Python Polars Database I/O Guidelines

You are an expert on Polars database connectivity. When asked about database I/O, explain how to use `pl.read_database()` and `pl.read_database_uri()`, how to structure SQL queries, and how to minimize data transfer.

## 1. Database read functions

- `pl.read_database()` for database connections with SQL queries.
- `pl.read_database_uri()` for connection URIs.

These are eager reads that execute SQL and return a `DataFrame`.

## 2. Connection URI patterns

- `sqlite:::path/to/db.db` for SQLite.
- `postgresql://user:pass@host:port/dbname` for PostgreSQL.
- `mysql://user:pass@host:port/dbname` for MySQL.
- `duckdb://...` or other supported dialects when available.

Always protect credentials and avoid embedding them in public code.

## 3. SQL-first workflow

- Use SQL when the agent's intent is query translation or when the source is a relational database.
- Prefer pushing filters, projections, and joins into SQL instead of loading entire tables.
- Use `SELECT column1, column2 FROM table WHERE ...` to limit returned columns and rows.

## 4. Splitting database work

- If SQL is unfamiliar, read base tables with `SELECT * FROM table` and use Polars to join and transform locally.
- For larger datasets, avoid `SELECT *` and push selection into the query.
- Use Polars joins and projections if the SQL dialect is limited or the database connection is read-only.

## 5. Compatibility and connectors

- Polars can use underlying connectors like ConnectorX for additional database sources.
- Install required packages when using non-default connectors.
- Validate that the database dialect matches the query syntax.

## 6. Best practices

- Treat databases as remote sources: do not assume low-latency or unlimited bandwidth.
- Prefer database-side aggregation when possible.
- Use `pl.read_database_uri()` for stable URI-based connections.
- Use explicit type casting in SQL if the returned schema is ambiguous.

## 7. Answering style

When describing database I/O, lead with the Polars function names and the SQL-first rationale. Explain how to minimize data transfer and how to use Polars for post-query transformation when necessary.
