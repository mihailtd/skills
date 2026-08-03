# Database — DuckDB

DuckDB *database* concepts: the in-process OLAP engine model, its columnar/vectorized
performance design, and SQL against files (CSV/Parquet) and in-memory DataFrames
without a load step. Code examples target the Python client (this repo's primary
language), but the engine itself is embeddable from many languages. Pairs with
Polars — see the `python` master skill's "Data Processing — Polars vs DuckDB"
decision heuristic.

## Install

```bash
npx skills add mihailtd/skills/skills/database-duckdb --all
```

Add `-g` to install globally instead of per-project. Use `--skill <name>` (repeatable) instead of `--all` to cherry-pick individual skills from this package, e.g.:

```bash
npx skills add mihailtd/skills/skills/database-duckdb --skill database-duckdb-overview
```

## Skills (20)

- **[database-duckdb-overview](database-duckdb-overview/SKILL.md)** — What DuckDB is (an in-process OLAP engine), why it eliminates the load-into-a-server step, its performance design pillars (columnar storage, vectorized execution, parallel execution, late materialization, optimized query planning), and a decision heuristic for DuckDB vs. Polars vs. pandas vs. PostgreSQL.
- **[database-duckdb-getting-started](database-duckdb-getting-started/SKILL.md)** — The core Python workflow: installing via `uv`, persistent/in-memory/read-only connections, defining tables, inserting records with parameterized queries, and querying/aggregating/joining data with results returned as Polars DataFrames.
- **[database-duckdb-cli](database-duckdb-cli/SKILL.md)** — The DuckDB command-line shell: launching against an in-memory or persistent database, useful startup flags, dot commands (`.tables`, `.schema`, `.open` vs `ATTACH`/`USE`, `.dump`, `.read`), and persisting an in-memory session with `EXPORT DATABASE`/`IMPORT DATABASE`.
- **[database-duckdb-ddl-crud](database-duckdb-ddl-crud/SKILL.md)** — Defining tables with `PRIMARY KEY`/`FOREIGN KEY` constraints, inspecting schema (`DESCRIBE`, `.schema`), dropping tables and the referential-integrity error, and the CRUD statements (`INSERT`/`UPDATE`/`DELETE`) including the unconditioned-`DELETE` danger.
- **[database-duckdb-querying-joins](database-duckdb-querying-joins/SKILL.md)** — Filtering with `WHERE` (including date/expression comparisons), all five join types and what each returns for unmatched rows, chaining joins across multiple tables, and why an unordered result needs an explicit `ORDER BY`.
- **[database-duckdb-aggregation](database-duckdb-aggregation/SKILL.md)** — `GROUP BY` and aggregate functions (`COUNT`/`AVG`/`MIN`/`MAX`), the `ORDER BY` + `LIMIT` top-N idiom and its silent-tie pitfall, and the scalar-subquery pattern for getting the full row behind an aggregate.
- **[database-duckdb-views-and-subqueries](database-duckdb-views-and-subqueries/SKILL.md)** — Date-arithmetic analytics with `DATEDIFF`, saving a reusable query with `CREATE VIEW`, and the nested-subquery-plus-`HAVING` pattern for surfacing every row tied for a maximum instead of silently picking one.
- **[database-duckdb-analytics-patterns](database-duckdb-analytics-patterns/SKILL.md)** — CTEs (`WITH`) for multi-stage aggregation, `CASE WHEN` bucketing (and DuckDB's `GROUP BY ALL` shortcut), the `SUM(CASE WHEN ...)`/`COUNT(*)` percentage-of-group idiom, `SIMILAR TO` data-quality checks, and why an implicit comma-join should become an explicit `JOIN`.
- **[database-duckdb-spatial-extension](database-duckdb-spatial-extension/SKILL.md)** — Geospatial analysis with the `spatial` extension: `ST_Point`/`ST_AsText` (and the longitude-first ordering gotcha), `ST_DWithin`/`ST_Distance` proximity queries, the degrees-aren't-real-distance caveat on WGS84 data, and briefly handing a result to a mapping/charting library.
- **[database-duckdb-dataframe-integration](database-duckdb-dataframe-integration/SKILL.md)** — Querying Polars DataFrames and CSV/Parquet files directly from SQL without a load step, the pyarrow bridge dependency, measuring the speed/memory tradeoff on your own data, and deciding when to scan in place versus materialize into a DataFrame first.
- **[database-duckdb-relational-api](database-duckdb-relational-api/SKILL.md)** — DuckDB's method-chaining `DuckDBPyRelation` API (`conn.table()`, `.filter()`, `.join()`, `.aggregate()`, `.project()`, `.limit()`, `.order()`, `.describe()`, `.insert()`) as a programmatic alternative to a SQL string, and when it beats one.
- **[database-duckdb-csv-io](database-duckdb-csv-io/SKILL.md)** — Loading CSV files (`read_csv_auto` vs explicit-schema `read_csv`/`COPY`), the `register()` method for binding a DataFrame as a queryable name, the header-auto-detection gotcha, and exporting query results back to CSV.
- **[database-duckdb-parquet-io](database-duckdb-parquet-io/SKILL.md)** — Reading and writing Parquet files (`read_parquet`, `COPY FROM`/`TO`), why Parquet's columnar layout pairs naturally with DuckDB, and when to convert a CSV dataset to it.
- **[database-duckdb-excel-io](database-duckdb-excel-io/SKILL.md)** — Reading and writing Excel spreadsheets via the `spatial` extension (`st_read`, the `OGR_XLSX_HEADERS`/`OGR_XLSX_FIELD_TYPES` environment variables), the explicit-schema pattern, and export caveats.
- **[database-duckdb-json-io](database-duckdb-json-io/SKILL.md)** — `read_json_auto` vs explicit-schema `read_json` (array/newline-delimited/unstructured formats), dot/bracket access into nested objects, `unnest()` for arrays under a key, multi-file loading and the cross-file schema-consistency gotcha, `COPY FROM`/`TO` for JSON.
- **[database-duckdb-httpfs](database-duckdb-httpfs/SKILL.md)** — Querying remote CSV/Parquet files directly by URL with the `httpfs` extension, why Parquet's metadata and HTTP range requests make remote access far cheaper than CSV, and inspecting a remote file's schema/row count without downloading its data.
- **[database-duckdb-huggingface-datasets](database-duckdb-huggingface-datasets/SKILL.md)** — Querying Hugging Face-hosted datasets via `hf://` paths (public datasets, folder-nested files, multi-file glob), persisting a remote dataset locally, and authenticating against a private dataset (`CREATE SECRET` `CONFIG` vs `CREDENTIAL_CHAIN`) — including which one is safe to commit.
- **[database-duckdb-external-databases](database-duckdb-external-databases/SKILL.md)** — Pulling data from a live PostgreSQL server into DuckDB with the `postgres` extension (`ATTACH`) versus a manual row-by-row driver loop, and why the extension is almost always the better default.
- **[database-duckdb-polars-postgresql-integration](database-duckdb-polars-postgresql-integration/SKILL.md)** — DuckDB vs. Polars (including Polars' own `SQLContext` SQL support and how it compares), DuckDB + Polars in one pipeline, DuckDB vs. PostgreSQL (OLAP engine vs. OLTP server), and DuckDB + PostgreSQL (offloading analytics off the transactional database) — the full comparison and combination guide.
- **[database-duckdb-marimo-notebooks](database-duckdb-marimo-notebooks/SKILL.md)** — Using DuckDB's SQL cells inside marimo notebooks (`mo.sql()`, custom connections, local-DataFrame/cross-cell references, UI-parameterized reactive queries), and the SQL-injection caveat that comes with f-string-interpolated queries.
