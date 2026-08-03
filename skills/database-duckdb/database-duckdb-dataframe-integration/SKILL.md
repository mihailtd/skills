---
name: database-duckdb-dataframe-integration

description: Guides the agent through querying Polars DataFrames and files (CSV/Parquet) directly from DuckDB SQL without a load/import step, measuring the speed and memory tradeoff against fully materializing data first, and deciding when to scan in place versus load into a Polars DataFrame.
---

# DuckDB DataFrame and File Integration

DuckDB's most distinctive capability isn't SQL syntax — it's that it can query data
that already lives somewhere else (a Polars DataFrame in memory, a CSV or Parquet
file on disk) without first importing or copying it. This skill covers that pattern
and when it's worth using instead of loading data into memory up front.

## 1. Querying a Polars DataFrame Directly by Name

Reference a local Polars DataFrame variable by name in a `FROM` clause — no
`CREATE TABLE`, no `INSERT`, no explicit registration step. DuckDB detects the
variable in the calling scope and scans it directly:

```python
import duckdb
import polars as pl

employees = pl.DataFrame({
    "employee_id": [1, 2, 3, 4],
    "name": ["Alice", "Bob", "Charlie", "David"],
    "department": ["HR", "Engineering", "Marketing", "Engineering"],
})
sales = pl.DataFrame({
    "sale_id": [101, 102, 103, 104, 105],
    "employee_id": [1, 2, 1, 3, 4],
    "sale_amount": [200, 500, 150, 300, 700],
})

conn = duckdb.connect()
conn.sql("""
    SELECT
        e.department,
        SUM(s.sale_amount) AS total_sales,
        AVG(s.sale_amount) AS average_sale_per_employee
    FROM employees e
    LEFT JOIN sales s ON e.employee_id = s.employee_id
    GROUP BY e.department
""").pl()
```

Both `employees` and `sales` stay exactly as they are — nothing is copied into a
DuckDB table before the join runs. This works the same way for pandas DataFrames
and Arrow tables, but Polars is the standard for DataFrame work in this repository
(see **python-polars-vs-pandas**), so default to it.

This interop goes through Apache Arrow. `duckdb` and `polars` both depend on it
already, but on some version combinations the conversion needs the `pyarrow`
package explicitly installed (`uv add "polars[pyarrow]"`) — if a query against a
local DataFrame fails with an Arrow-related import error, that's the first thing
to check.

`conn.sql(...)` isn't the only way to build a query against a DataFrame or table —
**database-duckdb-relational-api** covers `conn.table(...)` and method-chaining
(`.filter()`, `.join()`, `.aggregate()`, ...) as a programmatic alternative to a SQL
string, useful when a query is assembled conditionally rather than known upfront.

## 2. Scanning Files in Place

`read_csv_auto(...)` and `read_parquet(...)` query a file directly, inferring the
schema automatically, without materializing the whole file into memory first:

```python
conn.sql("""
    SELECT
        airline,
        AVG(arrival_delay) AS mean_arrival_delay
    FROM read_csv_auto('flights.csv')
    GROUP BY airline
    ORDER BY airline
""").pl()
```

This matters most for files that are large relative to available memory, or when
the query only needs a subset of columns/rows — DuckDB's columnar scan reads only
what the query actually needs, rather than paying the cost of fully loading a file
you're about to filter or aggregate down anyway.

## 3. Measuring the Difference on Your Own Data

Don't take a general performance claim on faith for a specific dataset and
machine — the size of the win depends on file size, column count, and how much
of the file the query actually touches. Verify directly:

```python
import time
import psutil


def memory_usage_mb() -> float:
    return psutil.Process().memory_info().rss / (1024**2)


before = memory_usage_mb()
start = time.perf_counter()
result = conn.sql("SELECT airline, AVG(arrival_delay) FROM read_csv_auto('flights.csv') GROUP BY airline").pl()
elapsed = time.perf_counter() - start
after = memory_usage_mb()

print(f"{elapsed:.2f}s, {after - before:.1f} MB delta")
```

Run the equivalent Polars-first approach (`pl.read_csv(...)` then `.group_by(...)`)
through the same measurement to compare on the actual dataset in question, rather
than assuming DuckDB's file-scanning path always wins — for a file that's re-read
and re-transformed many times in one session, loading once into a Polars DataFrame
can come out ahead.

## 4. Decision Guidance: Scan in Place vs. Materialize First

- **One-off query against a file you won't touch again in this session** → scan it
  directly with `read_csv_auto`/`read_parquet`. Paying to fully load it first buys
  you nothing.
- **File is large relative to available memory** → scan in place. Materializing it
  as a Polars DataFrame first risks memory pressure the columnar scan avoids.
- **Same data will go through many transformation steps** → load once into a
  Polars DataFrame (or a DuckDB table via **database-duckdb-getting-started**),
  then work against that — avoids DuckDB re-parsing/re-planning the source on
  every step.
- **Already have the data as a Polars DataFrame from earlier processing** → query
  it by name directly (section 1) rather than writing it out and reading it back.

## Related guidance

- **database-duckdb-overview** — the columnar/vectorized design that makes in-place scanning fast, and the broader DuckDB-vs-Polars decision.
- **database-duckdb-getting-started** — creating persistent tables when data does need to be loaded and kept.
- **python-polars-vs-pandas** — why this repository standardizes on Polars over pandas.
- **python-polars-csv-io** (package `python-polars`) — reading CSV files with Polars directly, the alternative to scanning them via DuckDB.
- **database-duckdb-marimo-notebooks** — this same DataFrame-by-name pattern used inside a marimo SQL cell.
- **database-duckdb-huggingface-datasets** — the same scan-once-vs-materialize tradeoff applied to a remote Hugging Face dataset.
- **database-duckdb-relational-api** — building a query against a DataFrame or table with chained method calls instead of a SQL string.
