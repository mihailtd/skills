---
name: database-duckdb-relational-api

description: Guides the agent through DuckDB's Relational API (DuckDBPyRelation) — building queries by chaining conn.table()/.filter()/.join()/.aggregate()/.project()/.limit()/.order()/.describe()/.insert() instead of writing a SQL string — and when method-chaining beats a plain conn.sql() call.
---

# DuckDB's Relational API (`DuckDBPyRelation`)

Every skill so far in this package builds queries as SQL strings passed to
`conn.sql()`/`conn.execute()`. DuckDB also has a second, programmatic way to build
the same queries: `conn.table(name)` returns a `DuckDBPyRelation` — a lazy handle
to a table (nothing runs until you materialize it) that you build up further by
chaining methods, each of which returns a new relation instead of running
anything. It's the same lazy-until-collected shape as a Polars `LazyFrame` or the
result of `conn.sql()` itself, just expressed as method calls instead of a SQL
string.

## 1. Getting a Relation

```python
import duckdb

conn = duckdb.connect()
conn.execute("CREATE TABLE customers (customer_id INTEGER PRIMARY KEY, name STRING)")
conn.execute("CREATE TABLE products (product_id INTEGER PRIMARY KEY, product_name STRING)")
conn.execute("""
    CREATE TABLE sales (
        customer_id INTEGER, product_id INTEGER, qty INTEGER,
        PRIMARY KEY (customer_id, product_id)
    )
""")

customers = conn.table("customers")
```

`conn.sql("SELECT * FROM customers")` returns the same kind of object
(`DuckDBPyRelation`) — `conn.table(...)` is just a shortcut for "give me the whole
table as a relation" without writing the trivial `SELECT *` yourself.

## 2. Inserting Rows Through a Relation

A relation bound directly to a table (from `conn.table(...)`, not a filtered/joined
derivative — see section 3) supports `.insert()`:

```python
customers.insert([1, "Alice"])
customers.insert([2, "Bob"])
customers.insert([3, "Charlie"])
```

This is a row-at-a-time insert, the same performance shape flagged in
**database-duckdb-external-databases** section 1 and **database-duckdb-excel-io**
section 3 — fine for a handful of rows, not the tool for bulk loading (use `COPY`
or a multi-row `INSERT ... VALUES`, per **database-duckdb-ddl-crud** section 4, for
volume).

## 3. Building a Query by Chaining

Each method below takes a SQL *fragment* as a string argument — not a full
statement, and not a typed expression object the way Polars' `pl.col(...)` is —
and returns a new relation you can keep chaining or branch from:

```python
result = customers.join(
    conn.table("sales"), condition="customer_id", how="inner"
).join(
    conn.table("products"), condition="product_id", how="inner"
)
```

`how` takes the same join types covered in **database-duckdb-querying-joins**
(`"inner"`, `"left"`, `"right"`, `"outer"`/full, `"cross"`) — this is method-call
syntax for the same joins, not a different join semantics.

```python
result.filter("customer_id = 1")                       # WHERE
result.aggregate("customer_id, MAX(name) AS name, SUM(qty) AS total_qty", "customer_id")  # SELECT ... GROUP BY
result.project("name, qty, product_name")               # SELECT (column list)
result.limit(3, 2)                                       # LIMIT 3 OFFSET 2
result.order("Year DESC")                                # ORDER BY
```

`aggregate(select_expr, group_expr)` is the relational-API equivalent of a
`SELECT <select_expr> FROM ... GROUP BY <group_expr>` statement — the two
arguments map directly onto those two SQL clauses.

## 4. Inspecting a Relation

`.describe()` returns summary statistics (min/max/median/count per column) as
another relation:

```python
customers.describe()
```

Materialize any relation into a Polars DataFrame with `.pl()` (or a pandas one
with `.df()` — see **database-duckdb-getting-started** for why this repository
defaults to `.pl()`), the same conversion used throughout this package.

## 5. Relational API vs. Plain SQL

Both build the exact same query plan under the hood — this is a syntax choice,
not a performance one. Reach for the relational API when a query is assembled
*conditionally* or *incrementally* in Python — e.g. adding a `.filter(...)` only
if a function argument is set, or building a shared base relation that several
call sites each extend differently. Reach for a plain SQL string via `conn.sql()`
(**database-duckdb-getting-started**) when the whole query is known upfront —
it's more readable as one block than as a chain of string-fragment method calls,
and it's what every other skill in this package uses by default.

## Related guidance

- **database-duckdb-getting-started** — `conn.sql()`, the plain-SQL-string alternative to this API.
- **database-duckdb-querying-joins** — the join types `how=` maps onto in section 3.
- **database-duckdb-ddl-crud** — bulk-insert alternatives to the row-at-a-time `.insert()` in section 2.
- **database-duckdb-dataframe-integration** — the `.pl()`/pyarrow interop this API's results go through the same way `conn.sql()` results do.
