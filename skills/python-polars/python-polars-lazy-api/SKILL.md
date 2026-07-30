---
name: python-polars-lazy-api
description: Instructs the agent on Polars lazy API usage with LazyFrame, deferred execution, query planning, and optimization strategies.
---

# Python Polars Lazy API Guidelines

You are an expert on the Polars lazy API. When asked about Polars workflows, explain `pl.LazyFrame` as the deferred execution model that enables query optimization and better performance on large or complex workloads.

## 1. Lazy execution model

- The lazy API builds a query plan without executing it immediately.
- Execution is deferred until `lf.collect()` or another terminal method is called.
- LazyFrames store a blueprint of operations instead of raw data.

Example:

```python
import polars as pl

lazy_df = pl.scan_csv("data/fruit.csv").with_columns(
    is_heavy=pl.col("weight") > 200
)
```

No data is loaded or transformed until `lazy_df.collect()` is invoked.

## 2. Use cases for the lazy API

Use lazy mode when:

- the dataset is large or remote
- you want optimizer-level query reductions
- the pipeline is complex and multi-step
- you need column pruning, predicate pushdown, or expression fusion
- you want to defer computation until the final result is known

## 3. Lazy API strengths

- Reduces unnecessary I/O by reading only needed columns and rows.
- Allows the optimizer to minimize the work performed.
- Often outperforms eager mode on analytic workloads.
- Can catch schema errors early via plan validation.
- Supports streaming and out-of-core execution via `collect(streaming=True)` and sink methods.

## 4. Lazy API weaknesses

- Less convenient for interactive inspection of intermediate results.
- Some DataFrame-only methods are unavailable.
- Repeated `collect()` calls can recompute the plan from scratch.
- The query plan may hide actual execution cost until runtime.

## 5. Lazy API best practices

### Build the pipeline first

Create a single chain of transformations and collect once.

```python
trips = pl.scan_parquet("data/taxi/yellow_tripdata_*.parquet")
result = (
    trips
    .group_by("VendorID")
    .sum()
    .select(
        "VendorID",
        income_per_distance=pl.col("total_amount") / pl.col("trip_distance"),
    )
    .sort(by="income_per_distance", descending=True)
    .head(3)
    .collect()
)
```

### Cache expensive intermediate results

Avoid re-running the same plan multiple times by caching materialized results:

```python
lf = expensive_lazy_plan.collect().lazy()
```

### Use explain and show_graph

- `lf.explain()` prints the optimized query plan.
- `lf.show_graph()` visualizes the query graph.

## 6. Lazy API error handling

- LazyFrames can fail early with schema errors before data is processed.
- Example: treating `i64` as `String` in a lazy expression will error on `collect()`.
- This fail-fast behavior is useful for large workloads because it avoids wasted compute.

## 7. Streaming and out-of-core execution

- `collect(streaming=True)` enables streaming materialization.
- `lf.sink_csv()`, `lf.sink_parquet()`, `lf.sink_ipc()`, and `lf.sink_ndjson()` write results to disk without fully loading them into memory.
- Streaming mode is experimental and should be used when datasets are larger than RAM.

## 8. Lazy compatibility rules

- Use `lf.collect()` to convert a `LazyFrame` to a `DataFrame`.
- Use `df.lazy()` to convert a `DataFrame` to a `LazyFrame`.
- Do not join a `DataFrame` directly with a `LazyFrame`; convert one side first.

Example:

```python
big_sales_data = pl.LazyFrame(...)
sales_metadata = pl.DataFrame(...)
joined = big_sales_data.join(sales_metadata.lazy(), on="sale_id").collect()
```

## 9. Lazy method categories

When describing lazy mode, the agent should refer to method categories such as:

- Attributes: `.columns`, `.dtypes`, `.schema`, `.width` (no `.shape`, `.height`, `.flags`)
- Aggregation: `.sum()`, `.mean()`, `.max()`, `.min()`, `.quantile()`, `.std()`, `.var()`
- Manipulation: `.select()`, `.filter()`, `.with_columns()`, `.join()`, `.group_by()`, `.sort()`, `.limit()`, `.head()`, `.tail()`
- Descriptive: `.explain()`, `.show_graph()`, `.collect_schema()`
- Miscellaneous: `.collect()`, `.collect_async()`, `.collect_schema()`, `.cache()`

## 10. Answering style

When responding, emphasize optimization benefits and deferred execution semantics. Give examples that contrast lazy plan building with eager materialization, and mention when `collect()` is the right time to finish the computation.
