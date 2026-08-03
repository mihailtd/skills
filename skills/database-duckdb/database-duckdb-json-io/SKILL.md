---
name: database-duckdb-json-io

description: Guides the agent through loading JSON into DuckDB — read_json_auto() vs explicit-schema read_json() (array/newline_delimited/unstructured formats), the records=false toggle, dot/bracket access into nested objects, unnest() for arrays under a key, loading multiple files (list or glob) and the cross-file schema-consistency gotcha, COPY FROM for high-volume simple JSON, and exporting with COPY TO (newline-delimited by default, ARRAY TRUE for a real JSON array).
---

# DuckDB JSON Import and Export

JSON has no fixed schema, so loading it is less mechanical than CSV or Parquet —
DuckDB has to either infer structure or be told it explicitly. This skill covers
both paths, plus the patterns needed once a file has nested objects, arrays, or
comes as several files at once.

## 1. Auto-Loading with `read_json_auto`

Try this first, always — it infers the schema and unpacks a JSON object's keys
into columns automatically:

```python
conn.execute("CREATE TABLE people AS FROM 'json1.json'")
conn.sql("SELECT * FROM people").pl()
```

`records` controls whether each JSON object becomes a row of separate columns
(`records=true`, the default) or a single struct-typed column per row
(`records=false`) — the latter is rarely what you want for tabular analysis, but
matters if you'd rather keep a record intact and pick fields out later:

```python
conn.sql("SELECT * FROM read_json_auto('json1.json', records=false)").pl()
```

Selecting specific top-level keys works exactly like any other query — no need to
load every field just to discard most of them:

```python
conn.sql("SELECT name, email FROM read_json_auto('json1.json')").pl()
```

## 2. Explicit Loading with `read_json`

`read_json_auto` is `read_json` with detection turned on — reach for `read_json`
directly, with an explicit `columns` map and `format`, when auto-detection gets a
file's structure wrong:

```python
conn.sql("""
    SELECT *
    FROM read_json('json1.json', format='auto', columns={
        id: 'INTEGER', name: 'STRING', weight: 'FLOAT'
    })
""").pl()
```

`format` is one of `array` (a single JSON array of objects), `newline_delimited`
(one object per line, no enclosing array — also accepted as `nd`),
`unstructured` (anything, including irregular JSON), or `auto` (detect it).
Omitting `columns` loads every key found, same as the auto function.

Newline-delimited JSON specifically has its own shortcut,
`read_ndjson_auto(...)`, which skips specifying `format` at all:

```python
conn.sql("""
    SELECT * FROM read_ndjson_auto('json1_a.json', columns={
        id: 'INTEGER', name: 'STRING', weight: 'FLOAT'
    })
""").pl()
```

## 3. Nested Objects — Dot and Bracket Access

A nested JSON object loads as a struct-typed column, not separate columns — index
into it with `['key']` (or `.key`, both work identically) to pull specific nested
fields out into their own columns:

```python
conn.sql("""
    SELECT
        id, name,
        address['line1'] AS line1,
        address['state'] AS state,
        address['zip'] AS zip,
        email, weight
    FROM read_json('json2.json')
""").pl()
```

Deeper nesting just chains the same access pattern — `address['location']['state']`
for a `location` object nested inside `address`. There's no depth limit to this;
it's the same struct-field access at every level.

## 4. Arrays Under a Key — `unnest()`

When the array of records you actually want is nested under a key (e.g. a
top-level `{"people": [...]}` wrapper rather than a bare top-level array),
`read_json` loads that key as a single struct-array column — `unnest(...)`
expands each array element into its own row:

```python
conn.sql("""
    SELECT
        p.id, p.name,
        p.address['line1'] AS line1,
        p.email, p.weight
    FROM (SELECT unnest(people) AS p FROM read_json('json3.json'))
""").pl()
```

Do the `unnest()` in a subquery (or CTE — see **database-duckdb-analytics-patterns**
section 1) first, then reference the unnested column's fields in the outer
`SELECT` — trying to both unnest and pull fields out in the same `SELECT` doesn't
work, since the fields don't exist as columns until after the unnest happens.

## 5. Loading Multiple Files

Pass a list of paths, or a glob pattern, to load several files as one result:

```python
conn.sql("SELECT * FROM read_json(['json4.json', 'json5.json'])").pl()
conn.sql("SELECT * FROM read_json('json*.json')").pl()   # glob: * ** ? [abc] [a-z]
```

**Schema-consistency gotcha:** rows are unioned by key name, not position — a
field missing from one file becomes `NULL`/`NaN` for that file's rows, which is
usually fine. A field with a *type* conflict across files (a string in one, a
number in another) is riskier: DuckDB may silently widen the column to a common
type (often casting everything to string) rather than erroring, so a numeric
column can quietly become text-typed with no warning. Before trusting a
multi-file load, check the inferred column types (`DESCRIBE`, per
**database-duckdb-ddl-crud** section 2) rather than assuming they came out the
way you expected — this is the same "verify before trusting" discipline as the
data-quality check in **database-duckdb-analytics-patterns** section 4, applied
to schema inference instead of column contents.

## 6. `COPY FROM` for High-Volume Simple JSON

For large files with a flat, well-known structure, `COPY FROM` into a pre-defined
table is faster than `read_json` (which loads the whole file into memory before
DuckDB can work with it) and integrates directly into a normal table-creation
flow:

```python
conn.execute("""
    CREATE TABLE people (id INTEGER, name VARCHAR, address VARCHAR, email VARCHAR, weight FLOAT);
    COPY people FROM 'json1.json' (FORMAT JSON, AUTO_DETECT true);
""")
```

This is the JSON analogue of the CSV `COPY` pattern in
**database-duckdb-csv-io** section 2 — same tradeoff: you supply the schema,
DuckDB does a fast bulk load against it. It doesn't handle nested structures well
(no path to express "unpack this nested object into these columns" the way
`read_json` can) — for anything beyond a flat object per record, use
`read_json`/`read_json_auto` instead, even at the cost of the full in-memory load.

## 7. Exporting to JSON

`COPY ... TO ... (FORMAT JSON)` writes a table out as **newline-delimited** JSON
by default — one object per line, not a single JSON array:

```python
conn.execute("COPY people TO 'people.json' (FORMAT JSON)")
```

For a genuine JSON array (`[{...}, {...}]`, valid as a single JSON document), add
`ARRAY TRUE`:

```python
conn.execute("COPY people TO 'people_array.json' (ARRAY TRUE)")
```

Pick based on the consumer: newline-delimited is the better fit for streaming/
line-oriented processing (and is itself something `read_ndjson_auto` reads back
directly); a true array is what's needed when something expects one parseable
JSON document.

## Related guidance

- **database-duckdb-csv-io** — the same auto-vs-explicit-schema and `COPY` tradeoffs, for CSV.
- **database-duckdb-ddl-crud** — `DESCRIBE` for checking inferred column types (section 5's gotcha) and the `CREATE TABLE` pattern `COPY FROM` builds on.
- **database-duckdb-analytics-patterns** — CTEs as the general tool for the "compute this first, use it in the next step" shape that `unnest()` needs in section 4.
- **database-duckdb-httpfs** / **database-duckdb-huggingface-datasets** — the same glob and multi-file schema-consistency concerns, applied to remote files.
