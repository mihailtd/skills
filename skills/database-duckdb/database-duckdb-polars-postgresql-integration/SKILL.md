---
name: database-duckdb-polars-postgresql-integration

description: Explains DuckDB versus Polars (and using them together via zero-copy Arrow handoff), DuckDB versus PostgreSQL (OLAP engine vs OLTP server), and DuckDB combined with PostgreSQL (offloading analytics off the transactional database, cross-source joins) — the full set of comparison and combination decisions for this repository's data stack.
---

# DuckDB, Polars, and PostgreSQL — Comparisons and Combinations

This repository uses three tools that all touch tabular data but solve different
problems: **Polars** (in-process DataFrame library), **DuckDB** (in-process SQL/OLAP
engine), and **PostgreSQL** (durable, multi-writer OLTP server). They aren't
alternatives to pick exactly one of — most real pipelines use two, sometimes all
three, each for the part it's actually good at. This skill covers all four angles:
DuckDB vs. Polars, DuckDB + Polars, DuckDB vs. PostgreSQL, and DuckDB + PostgreSQL.

## 1. DuckDB vs. Polars — Two Different Tools for the Same Data

Both are in-process, multithreaded, and built on Apache Arrow's columnar memory
model — neither needs a server, and both can outperform pandas by a wide margin on
the same hardware. The difference is the interface and what each optimizes for:

- **Query interface.** DuckDB is SQL-first, with a cost-based relational optimizer
  (predicate pushdown, join reordering) that reasons about a whole query at once.
  Polars is expression-first (`pl.col(...)` chains) with a lazy API that optimizes
  a chained pipeline the same way, but the two read very differently: SQL is
  often the more natural fit for ad hoc joins/aggregations and for anyone who
  reads SQL more fluently than a DataFrame API; Polars' expressions are the
  better fit for a pipeline with many small, named, reusable transformation
  steps and custom Python logic (`map_batches`, `map_elements`).
- **What they query.** DuckDB treats files (CSV/Parquet) and DataFrames as
  first-class queryable sources — it will join a Parquet file, a Polars
  DataFrame, and an attached PostgreSQL table in a single query without loading
  any of them fully first (see **database-duckdb-dataframe-integration** and
  section 4 below). Polars' native scanning is strong for its own file formats,
  but joining across genuinely heterogeneous sources — especially a live
  database — is DuckDB's strength, not Polars'.
- **Persistence.** A DuckDB connection can be a real, persistent database file
  with multiple tables, indexes, and a catalog you can reopen tomorrow. A Polars
  DataFrame is a transient in-memory object (or a file you explicitly wrote) —
  there's no notion of a Polars "database" to connect back to.
### Polars Has Its Own SQL Support Too — `SQLContext`

Polars can run SQL directly against its own DataFrames via `pl.SQLContext`,
without DuckDB in the picture at all:

```python
import polars as pl

cars = pl.DataFrame({
    "model": ["Camry", "Corolla", "RAV4"],
    "company": ["Toyota", "Toyota", "Toyota"],
    "engine_min": [2.5, 1.8, 2.0],
    "engine_max": [3.5, 2.0, 2.5],
})

ctx = pl.SQLContext(cars=cars)  # bind a name -> DataFrame, like a table
ctx.execute("SELECT * FROM cars", eager=True)

ctx.execute("""
    SELECT company, AVG(engine_min) AS avg_engine_min, AVG(engine_max) AS avg_engine_max
    FROM cars
    GROUP BY company
""", eager=True)
```

`eager=True` runs immediately and returns a `pl.DataFrame`; without it, `execute()`
returns a `LazyFrame` that only runs on `.collect()` — the same eager/lazy split
covered in the `python-polars` package's lazy-API skill, just entered through SQL
instead of the expression API.

This genuinely overlaps with what DuckDB does, and the two are easy to conflate:

| | `pl.SQLContext` | DuckDB (`conn.sql(...)`) |
|---|---|---|
| Scope | Only Polars DataFrames already bound into the context | Files, DataFrames, and attached databases (PostgreSQL, another DuckDB file) in the same query |
| SQL surface | A practical subset — common `SELECT`/`GROUP BY`/joins | A full, mature SQL engine — window functions, CTEs, `ATTACH`, extensions |
| Where the result lives | Stays inside Polars' own execution engine | DuckDB's engine; convert to Polars with `.pl()` when done |

Reach for `SQLContext` when the data is already Polars DataFrames, the query is
simple, and pulling in DuckDB as a dependency isn't otherwise justified. Reach for
DuckDB (this package) the moment the query needs to touch a file, an attached
database, or SQL DuckDB supports that Polars' subset doesn't — which, in practice,
is most non-trivial analytical work. Neither replaces
**database-duckdb-relational-api**'s method-chaining style, either — that's a
third, non-SQL way to build a query, specific to DuckDB.

See **database-duckdb-overview** section 4 for the quick decision table; this
section explains the *why* behind it.

## 2. DuckDB + Polars — Combining Them in One Pipeline

Because both are Arrow-based, moving data between them is cheap relative to
converting through a row-oriented structure (pandas, plain Python objects) —
there's no format translation, just handing off a columnar buffer. In practice
this means starting a pipeline in whichever tool fits the current step, and
switching mid-pipeline is normal, not a smell:

```python
import duckdb
import polars as pl

# 1. DuckDB: SQL-first ingestion/join across heterogeneous sources
raw = conn.sql("""
    SELECT e.department, s.sale_amount, s.sale_date
    FROM employees e
    LEFT JOIN read_parquet('sales_2025.parquet') s ON e.employee_id = s.employee_id
""").pl()

# 2. Polars: the feature-engineering / pipeline-chaining stage
featured = (
    raw.lazy()
    .with_columns(pl.col("sale_date").str.to_date())
    .group_by("department")
    .agg(pl.col("sale_amount").sum().alias("total_sales"))
    .collect()
)

# 3. Back to DuckDB for one step that's more natural as SQL (e.g. a window function
#    or a join against another attached source), referencing `featured` by name
conn.sql("SELECT * FROM featured ORDER BY total_sales DESC").pl()
```

The rule of thumb: use DuckDB for the parts of a pipeline that read from or join
across multiple sources (files, attached databases, other DataFrames), and Polars
for the parts that are a chain of programmatic transformations on data you already
have in hand. Neither tool needs to "win" the whole pipeline.

## 3. DuckDB vs. PostgreSQL — OLAP Engine vs. OLTP Server

These solve fundamentally different problems, not the same problem at different
speeds:

| | DuckDB | PostgreSQL |
|---|---|---|
| Workload shape | Few large analytical queries (scan, aggregate, join) | Many small transactional queries (point lookups, single-row writes) |
| Storage layout | Columnar | Row-oriented (with some columnar extensions) |
| Concurrency model | Single active writer per database; built for one analytical process, not many concurrent application clients | MVCC across many concurrent readers and writers — its core design goal |
| Access | In-process library, no network protocol | Client-server over the network, connection pooling, auth/roles |
| Durability tooling | No built-in replication or point-in-time recovery | Streaming/logical replication, WAL-based PITR, mature backup tooling |
| Indexing | Zone maps / min-max indexes suited to scan pruning | B-tree/GIN/GiST/BRIN suited to point lookups and varied query shapes |
| Role in this stack | Ad hoc analytics, file/DataFrame querying | System of record for application state (see the `python` master skill) |

Neither replaces the other in this repository: PostgreSQL is the durable,
multi-writer store the application talks to through SQLAlchemy
(`python-postgresql` package); DuckDB is a scratch analytical engine you reach for
when a question needs SQL and speed but not a running server or transactional
guarantees.

## 4. DuckDB + PostgreSQL — Offloading Analytics From the Transactional Database

The mechanics of attaching DuckDB to a live PostgreSQL server (the `postgres`
extension, `ATTACH`, querying attached tables directly) are covered in
**database-duckdb-external-databases**. The *why* comes down to two things that
combination buys you:

- **Isolating analytical load from transactional load.** A large aggregation run
  directly against production PostgreSQL competes with your application's own
  queries for the same connections, buffer cache, and CPU. Pulling the needed
  tables through DuckDB's `postgres` extension runs the heavy scan/aggregate work
  in a separate process against DuckDB's own columnar engine, instead of making
  the OLTP server do OLAP-shaped work it isn't optimized for.
- **Cross-source joins without a warehouse.** DuckDB can join an attached
  PostgreSQL table against a Parquet file, an S3 object, or a Polars DataFrame in
  a single query — something PostgreSQL alone would need a foreign data wrapper
  (heavier to set up) to do, and Polars alone can't do at all without first
  extracting the Postgres data some other way.

This is an analysis-time pattern, not an architecture change — the application
keeps talking to PostgreSQL exactly as it does today (see the `python` master
skill's database decision guide and the `python-postgresql` package); DuckDB sits
alongside it as a read-only analytical lens, not a replacement data layer.

## Putting It Together

| Situation | Reach for |
|---|---|
| Ad hoc SQL across a live PostgreSQL table and a Parquet/S3 file in one query | DuckDB, `postgres` extension attached |
| Complex multi-step feature-engineering pipeline with custom Python logic | Polars |
| Offloading a heavy aggregation off the production PostgreSQL server | DuckDB attached to Postgres, read-only |
| Need a DuckDB query's result as input to further DataFrame work | DuckDB → `.pl()` → Polars |
| Need one ad hoc SQL join/window function against an existing Polars DataFrame | Polars DataFrame → DuckDB (query by name) → `.pl()` back |
| Durable, transactional, multi-writer application state used by other services | **PostgreSQL** — not DuckDB, not Polars |

## Related guidance

- **database-duckdb-overview** — the quick decision table this skill expands on.
- **database-duckdb-dataframe-integration** — the mechanics of querying a Polars DataFrame or file directly from DuckDB.
- **database-duckdb-external-databases** — the mechanics of attaching a live PostgreSQL server.
- **database-duckdb-relational-api** — DuckDB's method-chaining query style, a third alternative alongside raw SQL and Polars' `SQLContext`.
- **python-polars-vs-pandas** — the separate (and settled) decision of Polars over pandas for in-process DataFrame work.
- **python-polars-lazy-api** (package `python-polars`) — the eager/lazy execution split `SQLContext`'s `eager=` argument maps onto.
- The `python` master skill (package `python-core`) — the "Data Processing — Polars vs DuckDB" heuristic and the PostgreSQL/MongoDB database decision guide.
- **database-duckdb-marimo-notebooks** — the DuckDB-then-Polars-then-DuckDB handoff pattern (section 2) expressed as marimo notebook cells instead of a linear script.
