---
name: python-polars-miscellaneous-methods
description: Instructs the agent on Polars miscellaneous methods for both DataFrame and LazyFrame, including collection, caching, profiling, and function mapping.
---

# Python Polars Miscellaneous Methods Guidelines

You are an expert on Polars miscellaneous methods. When asked about terminal or utility operations, explain the methods and their availability in `DataFrame` and `LazyFrame`.

## 1. Terminal collection methods

The primary terminal operations for `LazyFrame` are:

- `.collect(...)`
- `.collect_async()`
- `.collect_schema()`

These methods trigger execution and return either a materialized `DataFrame`, an async result, or just the schema.

## 2. Caching and reuse

- `.cache()` is used to store a result in memory for later reuse.
- In practice, cache a `LazyFrame` by calling `.collect().lazy()` after an expensive plan.

Example:

```python
lf = expensive_query.collect().lazy()
```

This avoids recomputing the same plan on subsequent operations.

## 3. Mapping and profiling

These utility methods are available in Polars:

- `.map_batches(...)`
- `.map_rows(...)`
- `.pipe(...)`
- `.profile(...)`
- `.corr(...)`
- `.equals(...)`

Explain that these are often used for advanced custom transformations, debugging, and performance measurement.

## 4. Lazy evaluation helper methods

- `.lazy()` converts a `DataFrame` to a `LazyFrame`.
- `.collect_schema()` returns the schema without loading all rows.

These are essential when moving between eager and lazy APIs.

## 5. Practical guidance

- Use `.profile()` to understand execution time and resource usage.
- Use `.map_batches(...)` for batch-wise transformations in a `DataFrame` or `LazyFrame` context.
- Use `.map_rows(...)` only when batch/columnar operations are insufficient.
- Use `.pipe(...)` to keep transformation chains readable.
- Use `.equals(...)` for deterministic equality checks on materialized frames.

## 6. Answering style

When asked about miscellaneous methods, provide the exact semantics: whether the method is terminal, plan-related, utility, or caching-related. Keep examples focused on how the method fits into a larger eager or lazy workflow.
