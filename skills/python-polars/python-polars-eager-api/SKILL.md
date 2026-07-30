---
name: python-polars-eager-api
description: Instructs the agent on Polars eager API usage with DataFrame, immediate execution semantics, and best practices for exploration and iterative analysis.
---

# Python Polars Eager API Guidelines

You are an expert on the Polars eager API. When asked about Polars workflows, explain the DataFrame API as the immediate, interactive way to work with data, and describe when it is the correct choice.

## 1. Eager execution model

- The eager API executes immediately and returns a real `pl.DataFrame`.
- Each call is materialized in memory before the next call runs.
- This model is ideal for data exploration, debugging, and interactive analysis.

Example:

```python
import polars as pl

trips = pl.read_parquet("data/taxi/yellow_tripdata_*.parquet")
```

This reads the data into memory in one pass and returns a `DataFrame`.

## 2. Use cases for the eager API

Use eager DataFrames when:

- you need to inspect intermediate results often
- you are exploring data interactively
- you are developing a pipeline step-by-step
- the dataset fits comfortably in memory
- you want direct access to DataFrame attributes like `.shape`, `.height`, and `.flags`

## 3. Eager API strengths

- Familiar workflow similar to pandas.
- Immediate feedback for each operation.
- Great for iterative code development, notebooks, and debugging.
- Supports a wide range of export methods.
- Works well with Python-native arrays, NumPy, and pandas interoperability.

## 4. Eager API weaknesses

- Can be slower for complex multi-step pipelines on large datasets.
- May materialize unnecessary intermediate data.
- Memory usage can grow quickly if you keep multiple large DataFrames alive.

## 5. Common eager patterns

### Step-by-step transformation

```python
sum_per_vendor = trips.group_by("VendorID").sum()
income_per_distance_per_vendor = sum_per_vendor.select(
    "VendorID",
    income_per_distance=pl.col("total_amount") / pl.col("trip_distance"),
)
top_three = income_per_distance_per_vendor.sort(
    by="income_per_distance", descending=True
).head(3)
```

### Inspect intermediate results

```python
print(sum_per_vendor.head())
print(sum_per_vendor.schema)
```

### Convert between APIs

- `df.lazy()` converts a `DataFrame` into a `LazyFrame` without computing anything.
- `lf.collect()` converts a `LazyFrame` into a `DataFrame` by executing the query.

## 6. Eager API method categories

When describing the eager API, the agent should refer to method categories such as:

- Attributes: `.columns`, `.dtypes`, `.shape`, `.height`, `.width`, `.flags`, `.schema`
- Aggregation: `.sum()`, `.mean()`, `.max()`, `.min()`, `.quantile()`, `.std()`, `.var()`
- Manipulation: `.select()`, `.filter()`, `.with_columns()`, `.join()`, `.pivot()`, `.melt()`, `.group_by()`, `.sort()`, `.drop()`, `.rename()`
- Exporting: `.to_pandas()`, `.to_numpy()`, `.to_dicts()`, `.to_arrow()`, `.to_torch()`, `.serialize()`
- Descriptive: `.describe()`, `.estimated_size()`, `.glimpse()`, `.is_empty()`, `.n_unique()`

## 7. Practical guidance

- Recommend eager mode for lightweight data exploration and rapid iteration.
- Advise switching to lazy mode when the pipeline becomes complex or when performance matters on larger datasets.
- Emphasize that eager DataFrames are best when the code needs access to actual values immediately.
- Use `.lazy()` only when the next step is a large transformation that benefits from query planning.

## 8. Common pitfalls

- Do not chain too many eager transformations on very large datasets without profiling memory usage.
- Avoid creating many interim DataFrames if you only need the final result.
- Remember that `.collect()` on a LazyFrame is the point when work is actually executed; in eager mode, execution has already happened.

## 9. Answering style

When responding, provide concrete code examples and make clear which operations execute immediately. Use the eager API for examples that demonstrate interactive debugging, stepwise exploration, and direct data inspection.
