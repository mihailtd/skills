---
name: database-duckdb-parquet-io

description: Guides the agent through reading and writing Parquet files with DuckDB (read_parquet, COPY FROM/TO, loading a slice with LIMIT/ORDER BY), why Parquet's self-describing columnar format pairs naturally with DuckDB's engine, and when to prefer it over CSV.
---

# DuckDB Parquet Import and Export

Parquet stores data column-by-column with the schema embedded in the file itself —
the same columnar layout that makes DuckDB's engine fast (see
**database-duckdb-overview**, section 2) applies at the storage layer too. A query
that only touches 2 of 30 columns reads only those 2 columns' bytes off disk; a CSV
read has no such option, since every field of every row is interleaved together.
That combination — self-describing schema, per-column compression, selective
column reads — is why Parquet is the standard hand-off format for analytical
workloads, data lakes, and cloud storage, and why it's worth converting a stable
CSV dataset to.

## 1. Loading Parquet Files

`read_parquet(...)` works the same way `read_csv_auto(...)` does for CSV:

```python
conn.execute("""
    CREATE TABLE airports AS
    SELECT * FROM read_parquet('airports.parquet')
    LIMIT 100
""")
```

To grab the *last* N rows instead of the first, sort descending by a column before
limiting — there's no dedicated "tail" operation, so this is the idiomatic way to
express it in SQL:

```python
conn.execute("""
    INSERT INTO airports
    SELECT * FROM read_parquet('airports.parquet')
    ORDER BY 1 DESC
    LIMIT 100
""")
```

To load a Parquet file into an already-defined table (schema enforced, rather than
inferred), use `COPY ... FROM`:

```python
conn.execute("COPY airports FROM 'airports.parquet' (FORMAT PARQUET)")
```

## 2. Writing Parquet Files

If the data is already a DuckDB table or query result, `COPY ... TO` with
`FORMAT PARQUET` writes it out directly — no extra library needed:

```python
conn.execute("COPY airports TO 'airports_all.parquet' (FORMAT PARQUET)")

# only a subset of rows
conn.execute("""
    COPY (SELECT * FROM airports LIMIT 100)
    TO 'airports_100.parquet' (FORMAT PARQUET)
""")
```

If the data is already a Polars DataFrame instead, Polars writes Parquet natively —
`df.write_parquet('file.parquet')` — no need to round-trip it through DuckDB, and
no need for a third-party Parquet-writing library either way; both DuckDB and
Polars have first-class Parquet support built in.

## 3. When to Prefer Parquet Over CSV

- **Repeated queries against the same dataset** — Parquet's columnar layout and
  embedded schema mean every subsequent read skips both type inference and
  reading columns the query doesn't need; CSV pays both costs every time.
- **Handing data off to another analytical tool** (Spark, cloud data lakes,
  another DuckDB/Polars process) — Parquet's self-describing schema avoids the
  "what are this CSV's actual column types" ambiguity entirely.
- **CSV still wins** for genuinely one-off interchange with a system that only
  speaks CSV, or human-inspectable output a non-technical consumer needs to open
  directly in a spreadsheet.

## Related guidance

- **database-duckdb-overview** — the columnar/vectorized design this format complements.
- **database-duckdb-csv-io** — the row-oriented interchange format Parquet is usually converted from.
- **database-duckdb-dataframe-integration** — querying a Parquet file in place with `read_parquet`, without a `CREATE TABLE` step.
- **database-duckdb-httpfs** — this format's local efficiency story, amplified further when the file is remote (metadata + HTTP range requests vs. downloading a whole CSV).
