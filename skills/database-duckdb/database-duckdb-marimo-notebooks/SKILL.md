---
name: database-duckdb-marimo-notebooks

description: Guides the agent through using DuckDB's SQL cells inside marimo notebooks (mo.sql(), custom connections, local-DataFrame/cross-cell references, reactive/UI-parameterized queries) — why this repo's notebook standard (marimo) and analytics engine (DuckDB) are built to work together, and the SQL-injection caveat that comes with f-string-interpolated queries.
---

# DuckDB Inside marimo Notebooks

This repository already standardizes on marimo over Jupyter for all notebooks (see
**python-notebooks-marimo**, package `python-core`). marimo's SQL cells are built
specifically around DuckDB's Python client, which makes the two a natural pairing
for interactive analysis: Python and SQL cells share the same reactive dataflow
graph, and a SQL cell is still stored as plain Python — no special notebook syntax,
no separate `.sql` files, no loss of the git-diffable-`.py`-file property that's the
whole reason this repo uses marimo in the first place.

## 1. Installing marimo with SQL Support

Install with the `sql` extra, and Polars with the `pyarrow` extra for DataFrame
interop — both via `uv`, not `pip`, per this repository's package-management rule:

```bash
uv add "marimo[sql]"
uv add "polars[pyarrow]"
```

`marimo tutorial sql` runs an interactive tutorial covering the same ground as this
skill, useful as a hands-on supplement.

## 2. Writing SQL Cells

Open a notebook with `marimo edit notebook.py`, then add a SQL cell (right-click the
`+` button and pick "SQL cell", convert an empty cell via its menu, or use the SQL
button at the bottom of the notebook). A SQL cell is serialized as ordinary Python
using `mo.sql()`:

```python
df = mo.sql(f"SELECT 'Off and flying!' AS a_duckdb_column")
```

The query string is an f-string — this is what lets a SQL cell depend on Python
values (UI elements, variables from other cells) as part of marimo's reactive graph,
covered in section 4. A `SELECT` assigns its result to `df` (a Polars DataFrame);
DDL statements like `CREATE TABLE` return `None`.

## 3. Custom Connections and Auto-Discovery

By default, SQL cells run against marimo's global in-memory DuckDB connection.
To use a specific connection instead — a persistent database file, or one with
extensions/attachments already configured (see
**database-duckdb-external-databases**) — create it as a plain Python variable in
a cell:

```python
import duckdb

conn = duckdb.connect("path/to/my/duckdb.db")
```

marimo detects `conn` automatically and offers it in the SQL cell's connection
dropdown. Once selected, marimo's Data Sources panel introspects that connection
and lists its databases, schemas, tables, and columns — useful for browsing an
unfamiliar schema without writing `SHOW TABLES`/`DESCRIBE` queries by hand.

## 4. Referencing DataFrames and Cross-Cell Results

A SQL cell can reference a local Polars DataFrame by the name of the Python
variable holding it — the same replacement-scan mechanism covered in
**database-duckdb-dataframe-integration**, just inside a marimo SQL cell instead
of a `conn.sql()` call:

```python
import polars as pl
df = pl.DataFrame({"column": [1, 2, 3]})
```

```sql
SELECT * FROM df WHERE column > 2
```

If a connected database has a table with the same name as a local DataFrame, the
database table takes precedence — name collisions resolve toward the database, not
the Python variable, so keep DataFrame names distinct from table names you also
query in the same notebook.

Assigning a SQL cell's result to a non-underscore-prefixed variable (`df = mo.sql(...)`,
not `_df = mo.sql(...)`) makes that result available to later cells, Python or SQL —
this is what lets a notebook chain a raw ingestion query into a Polars
transformation cell into a follow-up SQL cell, the same DuckDB-then-Polars-then-
DuckDB handoff described in **database-duckdb-polars-postgresql-integration**
section 2, expressed as marimo cells instead of a linear script.

## 5. Reactive, UI-Parameterized SQL Cells

Because a SQL cell is an f-string, interpolating a marimo UI element's value into
the query makes the query itself reactive — changing the slider re-runs every cell
that depends on the query, including the SQL cell:

```python
digits = mo.ui.slider(label="Digits", start=100, stop=10000, step=200)
digits
```

```sql
CREATE TABLE random_data AS
    SELECT i AS id, random() AS random_value
    FROM range({digits.value}) AS t(i);
```

For expensive queries over large datasets, configure marimo's runtime to lazy
execution — dependent cells are marked stale instead of re-running immediately,
so you control when the next (possibly costly) run actually happens rather than
paying for it on every slider tick.

**SQL-injection caveat:** f-string interpolation is safe here because `digits.value`
comes from a constrained UI element (a slider bounded to a numeric range) — the
value space is controlled by the notebook, not by arbitrary user input. The same
pattern is **not** safe for a free-text UI element (`mo.ui.text()`) whose value gets
interpolated straight into a query string — that's the same SQL-injection surface
as any other string-built query, called out already in
**database-duckdb-getting-started** section 3. Validate/allowlist a free-text value
before interpolating it, or bind it as a parameter through a plain
`conn.execute(sql, params)` call instead of `mo.sql()`'s f-string form when the
input is genuinely free text.

## 6. Why the Combination Works

- **One dataflow graph, two languages.** Python and SQL cells participate in the
  same reactive graph — a change anywhere re-runs exactly what depends on it,
  the property that makes marimo notebooks correct-by-construction (see
  **python-notebooks-marimo**) extends across the SQL/Python boundary instead of
  stopping at it.
- **UI-parameterized analytics.** Sliders, dropdowns, and text inputs can drive a
  SQL query directly, turning a static analysis into something interactively
  explorable without a separate app-building framework.
- **Still just a `.py` file.** A SQL cell is `mo.sql(f"...")` under the hood — the
  notebook stays a single, git-diffable Python file, runnable as a script, exactly
  as any other marimo notebook in this project.
- **Same DuckDB, same export story.** Everything else in this package (file/
  DataFrame querying, extensions, attached databases) works identically inside a
  marimo SQL cell as it does in a plain script — marimo changes how you write and
  run the notebook, not what DuckDB itself can do.

## Related guidance

- **python-notebooks-marimo** (package `python-core`) — the general marimo-over-Jupyter standard this skill builds on.
- **database-duckdb-getting-started** — connections and the parameterized-query security note referenced in section 5.
- **database-duckdb-dataframe-integration** — the local-DataFrame replacement-scan mechanism used in section 4.
- **database-duckdb-external-databases** — attaching a persistent/external connection to use via the custom-connection pattern in section 3.
- **database-duckdb-polars-postgresql-integration** — the DuckDB-then-Polars-then-DuckDB handoff pattern referenced in section 4.
