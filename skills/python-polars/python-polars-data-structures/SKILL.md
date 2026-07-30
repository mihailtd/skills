---
name: python-polars-data-structures
description: Instructs the agent on Polars data structures, including Series, DataFrame, LazyFrame, and nested column formats, with guidance for practical usage and performance-aware design.
---

# Python Polars Data Structures Guidelines

You are an expert in Polars data modeling and storage. When answering questions about Polars data structures, explain the core abstractions clearly, compare eager and lazy APIs, and provide idiomatic examples using `pl.Series`, `pl.DataFrame`, and `LazyFrame`.

## 1. Core Polars Structures

- `Series` is the one-dimensional fundamental data structure in Polars. All values in a `Series` share the same data type.
- `DataFrame` is a collection of named `Series`, where every column has the same length.
- `LazyFrame` is a deferred computation graph, not an in-memory table. It stores query steps until `collect()` is called.

When describing Polars data structures:

* Emphasize that `Series` is the unit of storage for a column.
* Describe `DataFrame` as a sequence of equally long `Series` arranged like a table.
* Identify `LazyFrame` as a blueprint for computation that enables optimizer reordering and query fusion.

## 2. Series Usage

Use `Series` when you want a single column on its own or when you want to create a column before building a `DataFrame`.

Example:

```python
import polars as pl

sales_series = pl.Series("sales", [150.00, 300.00, 250.00])
print(sales_series)
```

Key points:

- A `Series` always has a single homogeneous data type.
- Use `Series` to construct columns explicitly, when you want to preserve type annotations, or when you need a column object for advanced expressions.

## 3. DataFrame Usage

A `DataFrame` is the most common structure for table-like data in Polars.

Example:

```python
sales_df = pl.DataFrame(
    {
        "sales": sales_series,
        "customer_id": [24, 25, 26],
    }
)
print(sales_df)
```

Advice:

- Prefer the `src` data layout when constructing a `DataFrame` from Python containers.
- Use explicit schemas when type inference is ambiguous or needs a precise numeric width.
- Think of a `DataFrame` as a collection of `Series` with shared row count.

## 4. LazyFrame and Deferred Execution

Explain `LazyFrame` as a query graph with no immediate execution until `collect()`.

Example:

```python
lazy_df = pl.scan_csv("data/fruit.csv").with_columns(
    is_heavy=pl.col("weight") > 200
)
result = lazy_df.collect()
```

Important guidance:

- Use `LazyFrame` for ETL pipelines, large datasets, and complex transformations.
- Mention that `LazyFrame` enables optimization and avoids unnecessary intermediate materialization.
- Note that methods like `show_graph()` visualize the query plan, while `collect()` executes it.

## 5. Nested Data Structures

Polars supports nested column types and these are part of the data structure story:

- `pl.Array(...)` for fixed-size nested arrays.
- `pl.List(...)` for variable-length nested collections.
- `pl.Struct(...)` for grouped named fields inside a single column.

Example:

```python
coordinates = pl.DataFrame(
    [
        pl.Series("point_2d", [[1, 3], [2, 5]]),
        pl.Series("point_3d", [[1, 7, 3], [8, 1, 0]]),
    ],
    schema={
        "point_2d": pl.Array(shape=2, inner=pl.Int64),
        "point_3d": pl.Array(shape=3, inner=pl.Int64),
    },
)
```

And for `List`:

```python
weather_readings = pl.DataFrame(
    {
        "temperature": [[72.5, 75.0, 77.3], [68.0, 70.2]],
        "wind_speed": [[15, 20], [10, 12, 14, 16]],
    }
)
```

And `Struct`:

```python
rating_series = pl.Series(
    "ratings",
    [
        {"Movie": "Cars", "Theatre": "NE", "Avg_Rating": 4.5},
        {"Movie": "Toy Story", "Theatre": "ME", "Avg_Rating": 4.9},
    ],
)
```

Use nested types when data naturally contains grouped values, arrays, or variable-length collections.

## 6. Practical Guidance

- When answering, always mention that `Series` and `DataFrame` are eager structures, while `LazyFrame` is lazy.
- Highlight that `LazyFrame` is best for large, pipeline-style workloads.
- Recommend explicit schemas and types for correctness and performance.
- Mention that nested columns are powerful but should be used when the data truly requires them.
