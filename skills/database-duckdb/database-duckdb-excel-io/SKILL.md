---
name: database-duckdb-excel-io

description: Guides the agent through reading and writing Excel spreadsheets with DuckDB's spatial extension (INSTALL/LOAD, st_read with the layer argument, the OGR_XLSX_HEADERS/OGR_XLSX_FIELD_TYPES environment variables), the explicit-schema pattern for controlling column names/types, and the export caveats (no COPY support, no date/timestamp columns).
---

# DuckDB Excel Import and Export

DuckDB has no native Excel reader — support comes through the `spatial` extension,
which wraps GDAL/OGR (a geospatial data library that happens to also read Excel).
This is a heavier dependency than `read_csv`/`read_parquet` need, so reach for it
specifically when the data genuinely arrives as `.xlsx` and converting it to CSV
first isn't an option.

## 1. Installing the Extension

Install once per DuckDB installation — it's remembered across sessions after that,
so this doesn't need to run on every connection, only after a fresh install:

```python
conn.execute("INSTALL spatial")
conn.execute("LOAD spatial")
```

DuckDB has several other extensions worth knowing about for similar reasons:
`httpfs` (reading/writing over HTTP and cloud storage), `icu` (Unicode-aware string
processing), `sqlite` (querying SQLite files directly), `inet` (IP address/network
types). Install only what a given task actually needs.

## 2. Reading a Worksheet

`st_read(path, layer=...)` reads one worksheet by name:

```python
conn.execute("""
    CREATE TABLE airports AS
    SELECT * FROM st_read('airports_and_airlines.xlsx', layer='airports')
""")
```

Password-protected workbooks aren't supported — decrypt before loading.

By default the first row is auto-detected as a header if it looks like one. When a
worksheet genuinely has no header row, DuckDB falls back to generic `Field1`,
`Field2`, ... names. Control this explicitly with the `OGR_XLSX_HEADERS`
environment variable (`AUTO` is the default; `FORCE` always treats row 1 as headers,
`DISABLE` never does):

```python
import os
os.environ["OGR_XLSX_HEADERS"] = "DISABLE"
```

## 3. Applying a Custom Schema

When a worksheet has no header (or you don't trust auto-detection), define the
table's real column names and types up front, then load into it with `INSERT`
rather than `CREATE TABLE ... AS`:

```python
conn.execute("""
    CREATE TABLE airlines (iata_code VARCHAR, airline VARCHAR);
    INSERT INTO airlines
        SELECT * FROM st_read('airports_and_airlines.xlsx', layer='airlines');
""")
```

`COPY` doesn't work with Excel sources — `INSERT ... SELECT` (as above) is the only
load path, and it's row-oriented rather than the bulk-optimized path `COPY` gives
CSV/Parquet. This is fine for a single worksheet load; don't reach for a
row-by-row Python loop on top of it (see **database-duckdb-external-databases**
section 1 for why that combination is specifically the pattern to avoid for larger
volumes).

To force every column to load as a string regardless of its apparent type, set
`OGR_XLSX_FIELD_TYPES` to `STRING` (default `AUTO`):

```python
os.environ["OGR_XLSX_FIELD_TYPES"] = "STRING"
```

## 4. Exporting to Excel

```python
conn.execute("""
    COPY airlines TO 'airlines.xlsx' WITH (FORMAT GDAL, DRIVER 'xlsx')
""")
```

Two caveats: this errors if the destination file already exists (no implicit
overwrite), and the `xlsx` writer doesn't support `DATE`/`TIMESTAMP` columns —
cast them to `VARCHAR` before exporting.

## Related guidance

- **database-duckdb-csv-io** — the simpler path when converting to CSV first is an option; no extension required.
- **database-duckdb-external-databases** — the same "don't do it row-by-row" performance principle applied to loading from a live database server instead of a spreadsheet.
- **database-duckdb-spatial-extension** — the same `spatial` extension's actual geospatial functions (`ST_Point`, `ST_DWithin`, `ST_Distance`), distinct from its `st_read` use here.
