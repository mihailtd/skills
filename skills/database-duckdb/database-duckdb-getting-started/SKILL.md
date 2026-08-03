---
name: database-duckdb-getting-started

description: Guides the agent through the core DuckDB Python workflow — installing via uv, choosing a persistent vs in-memory connection, defining tables, inserting records with parameterized queries, and querying/aggregating/joining data with results returned as Polars DataFrames.
---

# DuckDB Getting Started (Python)

This skill covers the mechanical basics of working with DuckDB from Python: opening
a connection, defining a schema, inserting and querying data, and getting results
back as a Polars DataFrame. See **database-duckdb-overview** first if you haven't
already, for when DuckDB is the right tool to reach for.

## 1. Installing and Connecting

Install the `duckdb` package the same way as any other dependency in this
repository — with `uv`, not `pip`:

```bash
uv add duckdb
```

`duckdb.connect()` opens a connection. Pass a file path for a **persistent**
database, or `:memory:` (or no argument) for an **in-memory** one:

```python
import duckdb

# persistent — data survives process restarts, written to disk
conn = duckdb.connect("analytics.duckdb")

# in-memory — fast, but everything is lost when the connection closes;
# fine for a scratch analysis, wrong choice for anything you need to keep
conn = duckdb.connect(":memory:")
```

Use the persistent form whenever the data or the computed results need to outlive
the process. Use the in-memory form for a one-off exploration, a test fixture, or
any scratch query you don't need to keep.

Pass `read_only=True` when multiple processes need to open the same persistent
database file concurrently — DuckDB's file locking otherwise allows only one
read/write connection to a given file at a time. `read_only` only makes sense on
an existing persistent file; leave it `False` (the default) for in-memory
connections, since a read-only in-memory database can never have anything
attached to it in the first place:

```python
conn = duckdb.connect("analytics.duckdb", read_only=True)
```

## 2. Defining Tables

Pass DDL to `conn.execute()`:

```python
conn.execute("""
    CREATE TABLE employees (
        id INTEGER PRIMARY KEY,
        name VARCHAR,
        age INTEGER,
        department VARCHAR
    )
""")
```

Inspect the schema with `conn.sql(...)`, which returns a relation you can convert
to a Polars DataFrame with `.pl()`:

```python
conn.sql("SHOW TABLES").pl()
```

## 3. Inserting Records

For literal, trusted values, a multi-row `INSERT INTO ... VALUES` is fine:

```python
conn.execute("""
    INSERT INTO employees VALUES
        (1, 'Alice', 30, 'HR'),
        (2, 'Bob', 35, 'Engineering'),
        (3, 'Charlie', 28, 'Marketing'),
        (4, 'David', 40, 'Engineering')
""")
```

When any value comes from outside the program (user input, an API payload), use
`?` placeholders and pass parameters separately — never interpolate untrusted
values into the SQL string, the same rule as any other SQL engine:

```python
def add_employee(conn: duckdb.DuckDBPyConnection, emp_id: int, name: str, age: int, department: str) -> None:
    conn.execute(
        "INSERT INTO employees VALUES (?, ?, ?, ?)",
        [emp_id, name, age, department],
    )
```

## 4. Querying, Aggregating, and Converting Results

Use `conn.sql(...)` and convert the result with `.pl()` — this repository standardizes
on Polars, not pandas, for DataFrame work (see **python-polars-vs-pandas**), and
DuckDB's Python client supports both equally:

```python
conn.sql("SELECT * FROM employees").pl()
```

Aggregation is ordinary SQL — `GROUP BY` with `COUNT`, `AVG`, `MAX`, `SUM`:

```python
conn.sql("""
    SELECT department, COUNT(*) AS employee_count
    FROM employees
    GROUP BY department
""").pl()

conn.sql("""
    SELECT department, AVG(age) AS average_age, MAX(age) AS oldest_age
    FROM employees
    GROUP BY department
""").pl()
```

`conn.sql(...)` returns a lazy `DuckDBPyRelation` — nothing runs until you call
`.pl()`, `.df()`, `.arrow()`, or `.fetchall()` on it, similar in spirit to Polars'
own lazy API. `conn.execute(...)` is the alternative, DB-API-style entry point;
prefer it specifically when you need parameter binding (as in the insert example
above), and `conn.sql(...)` otherwise.

## 5. Joining Tables

Joins and multi-table queries are unremarkable standard SQL:

```python
conn.execute("CREATE TABLE orders (order_id INTEGER, customer_id INTEGER, amount FLOAT)")
conn.execute("""
    INSERT INTO orders VALUES (1, 1, 100.0), (2, 2, 200.0), (3, 1, 150.0)
""")

conn.execute("CREATE TABLE customers (customer_id INTEGER, name VARCHAR)")
conn.execute("INSERT INTO customers VALUES (1, 'Alice'), (2, 'Bob')")

conn.sql("""
    SELECT
        c.customer_id,
        c.name,
        SUM(o.amount) AS total_spent
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    GROUP BY c.customer_id, c.name
    ORDER BY c.customer_id
""").pl()
```

## Related guidance

- **database-duckdb-overview** — when DuckDB is the right tool versus Polars, pandas, or PostgreSQL.
- **database-duckdb-cli** — the same connection concepts (persistent vs in-memory vs read-only) from the command-line shell instead of Python.
- **database-duckdb-ddl-crud** — table constraints (PRIMARY KEY/FOREIGN KEY) and the full CRUD statement set, one level deeper than this skill's basics.
- **database-duckdb-querying-joins** — filtering and the full set of join types, beyond this skill's single-join example.
- **database-duckdb-dataframe-integration** — querying Polars DataFrames and files directly, without a `CREATE TABLE`/`INSERT` step.
- **python-polars-vs-pandas** — why results are converted with `.pl()`, not `.df()`, in this repository.
- **database-duckdb-marimo-notebooks** — the parameterized-query security note in section 3, revisited for marimo's f-string SQL cells.
