---
name: database-duckdb-csv-io

description: Guides the agent through loading CSV files into DuckDB (read_csv_auto vs read_csv with explicit schema, the COPY statement for best-performance bulk loads, the register() method for binding a DataFrame as a queryable name, and the header-auto-detection gotcha), plus exporting query results back to CSV.
---

# DuckDB CSV Import and Export

CSV remains the most common interchange format, and DuckDB gives you several ways
to load one — pick based on how much control you need over the resulting schema,
not habit. See **database-duckdb-dataframe-integration** first if you just want to
query a CSV file directly without ever materializing a table from it.

## 1. Loading CSV Files with SQL

`read_csv_auto(...)` infers the delimiter, header, and column types:

```python
conn.execute("""
    CREATE OR REPLACE TABLE flights AS
    SELECT * FROM read_csv_auto('flights.csv')
""")
```

`SELECT * FROM read_csv_auto(...)` and the bare `FROM 'flights.csv'` shorthand are
equivalent — DuckDB infers `read_csv_auto` for a bare file path:

```python
conn.execute("CREATE OR REPLACE TABLE flights AS FROM 'flights.csv'")
```

Prefer `CREATE OR REPLACE TABLE` over `DROP TABLE IF EXISTS` followed by
`CREATE TABLE` — same effect, one statement instead of two. When previewing a
large file, add `LIMIT` rather than loading the whole thing to check the schema:

```python
conn.execute("""
    CREATE OR REPLACE TABLE flights AS
    FROM read_csv_auto('flights.csv')
    LIMIT 1000
""")
```

## 2. Explicit Schema and Column Control

Auto-detection is convenient but not always right — a malformed header, or a header
row that looks like data, can fool it (see the gotcha in section 3). When you need
a specific schema, define the table first and load with `COPY`, which is also the
fastest bulk-load path for large files:

```python
conn.execute("""
    CREATE TABLE airports (
        iata_code VARCHAR, airport VARCHAR, city VARCHAR,
        state VARCHAR, country VARCHAR, latitude VARCHAR, longitude VARCHAR
    );
    COPY airports FROM 'airports.csv' (AUTO_DETECT TRUE);
""")
```

The CSV's column count must match the table's column count, and every value must
be convertible to its column's declared type, or the load fails. An alternative that
skips the manual `CREATE TABLE` step is passing `names` directly to `read_csv`:

```python
conn.execute("""
    CREATE OR REPLACE TABLE airports AS
    FROM read_csv('airports.csv', names=['iata_code', 'airport', 'city', 'state', 'country', 'latitude', 'longitude'])
""")
```

If you want every column typed as a string regardless of what's actually in the
file — useful for a first-pass load you'll cast from later — pass `all_varchar=true`
to `read_csv`. To confirm how many columns actually landed on a table:

```python
conn.sql("""
    SELECT COUNT(*) AS column_count
    FROM information_schema.columns
    WHERE table_name = 'airports'
""").pl()
```

## 3. Loading Without a Table — `register()` and the Header-Detection Gotcha

`read_csv_auto` can misread a file's header row as data (or vice versa). When that
happens, be explicit with `read_csv`'s `header` and `columns` arguments instead of
relying on detection:

```python
airlines = conn.sql("""
    SELECT * FROM read_csv(
        'airlines.csv',
        header = true,
        columns = {'iata_code': 'VARCHAR', 'airline': 'VARCHAR'}
    )
""").pl()
```

Always sanity-check what `read_csv_auto` actually produced against the source file
before trusting it in a pipeline — this behavior can change between DuckDB releases.

`conn.register(name, obj)` binds a Polars DataFrame (or pandas DataFrame, or Arrow
table) to a name in the connection's catalog, queryable like any table:

```python
conn.register("airlines", airlines)
conn.sql("SELECT * FROM airlines").pl()
```

This is a different mechanism from the automatic local-variable detection covered
in **database-duckdb-dataframe-integration** — `register()` explicitly binds a name
that persists across multiple `execute()`/`sql()` calls on that connection, which
matters when the object isn't a plain local variable DuckDB's replacement scan can
see (e.g. a dict/list element, or an object you want queryable under a different
name than its Python variable).

## 4. Exporting to CSV

`COPY (query) TO 'file.csv' WITH (...)` writes a query result to CSV:

```python
conn.execute("""
    COPY (SELECT iata_code, latitude, longitude FROM airports)
    TO 'airports_location.csv' WITH (HEADER 1, DELIMITER ',')
""")
```

The source of the `COPY` doesn't have to be a DuckDB table — you can read directly
from a file, filter/limit it, and write the result out, without ever materializing
the source into a table:

```python
conn.execute("""
    COPY (SELECT iata_code, latitude, longitude FROM 'airports.csv' LIMIT 10)
    TO 'airports_location.csv' WITH (HEADER 1, DELIMITER ',')
""")
```

## Related guidance

- **database-duckdb-getting-started** — connections, basic table CRUD.
- **database-duckdb-dataframe-integration** — querying a CSV or DataFrame in place without a `register()`/table step at all.
- **database-duckdb-parquet-io** — the columnar alternative to CSV, worth converting to once a dataset stabilizes.
- **database-duckdb-json-io** — the same auto-vs-explicit-schema and `COPY` tradeoffs, applied to JSON's nested/variable structure.
