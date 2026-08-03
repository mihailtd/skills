---
name: database-duckdb-httpfs

description: Guides the agent through DuckDB's httpfs extension for querying remote CSV/Parquet files directly by URL — installing/loading, filtering a remote file without downloading it locally first, reading a list of remote files, why Parquet's metadata and HTTP range requests make remote access dramatically cheaper than CSV, and inspecting a remote file's schema/row count without downloading its data.
---

# DuckDB `httpfs` — Querying Remote Files

Everything so far in this package has queried local files. `httpfs` extends the same
`FROM 'path'` pattern to `http://`/`https://` URLs — DuckDB reads the remote file
directly, without a separate download-then-load step.

## 1. Installing and Loading

```python
conn.execute("INSTALL httpfs")
conn.execute("LOAD httpfs")
```

`httpfs` is autoloadable — DuckDB will install/load it automatically the first time
a query touches a remote path, even without this step. Run it explicitly anyway in
scripts and CI, where autoloading may be disabled or where making the dependency
visible matters for reproducibility.

## 2. Querying a Remote File Directly

A URL works anywhere a local path would — no separate download step, and
ordinary `WHERE`/filtering runs against the remote file the same as a local one:

```python
url = "https://raw.githubusercontent.com/example/repo/main/AMZN.csv"
conn.sql(f"SELECT * FROM '{url}' WHERE year(Date) = 2018").pl()
```

`read_csv([...])` accepts a list of URLs, concatenating them the same way it does
for local files (see **database-duckdb-json-io** section 5's multi-file loading) —
and carries the identical schema-consistency requirement: files with different
schemas won't concatenate row-wise the way you'd expect, so verify column names
and types line up before relying on the combined result.

## 3. Why Parquet Is Dramatically Cheaper to Query Remotely

A remote **CSV** query downloads the *entire* file first — CSV is row-oriented, so
there's no way to fetch "just the columns this query needs" without reading every
byte. A remote **Parquet** query is different: DuckDB reads the file's embedded
metadata plus targeted HTTP range requests to pull down only the row groups and
columns the query actually touches — the same columnar-scan efficiency covered
in **database-duckdb-overview** section 2, now operating over the network instead
of local disk. For anything beyond a one-off small file, prefer Parquet sources for
remote data specifically because of this — the local-vs-remote gap in cost is much
larger than the local-only comparison in **database-duckdb-parquet-io**.

## 4. Inspecting a Remote Schema Without Downloading

`DESCRIBE TABLE 'url'` fetches just the schema — useful for deciding which
columns are worth pulling down before running the real query:

```python
url = "https://example.com/travel_insurance.parquet"
conn.sql(f"DESCRIBE TABLE '{url}'").pl()
```

## 5. Column Pruning and Metadata-Only Queries

Select only the columns a task needs — DuckDB fetches just those columns' data
from the remote Parquet file, not the whole thing:

```python
conn.sql(f"SELECT Agency, \"Agency Type\" FROM '{url}'").pl()
conn.sql(f"SELECT AVG(age) FROM '{url}'").pl()
```

`COUNT(*)` on a remote Parquet file can be answered from the file's metadata
alone — no row data needs to move over the network at all for that specific query
shape, the cheapest possible remote query.

## Related guidance

- **database-duckdb-overview** — the columnar/vectorized design this skill's remote-access efficiency builds on.
- **database-duckdb-parquet-io** — the local-file version of the CSV-vs-Parquet efficiency argument in section 3.
- **database-duckdb-json-io** — the same multi-file schema-consistency requirement, for JSON instead of CSV.
- **database-duckdb-huggingface-datasets** — a specific, structured remote-file scheme (`hf://`) built on the same underlying mechanics as this skill.
