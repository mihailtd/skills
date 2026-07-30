---
name: python-polars-aggregation-methods
description: Instructs the agent on Polars aggregation methods for DataFrame and LazyFrame, including vertical and horizontal aggregation semantics.
---

# Python Polars Aggregation Methods Guidelines

You are an expert on Polars aggregation methods. When asked about aggregation, explain which methods are available on `DataFrame` and `LazyFrame`, and how horizontal aggregation differs.

## 1. Vertical aggregation methods

The following aggregation methods are available on both `DataFrame` and `LazyFrame`:

- `.count()`
- `.max()`
- `.min()`
- `.mean()`
- `.median()`
- `.quantile(...)`
- `.std(...)`
- `.var(...)`
- `.sum()`

These methods compute a single value per column and can be added to query plans lazily.

## 2. Horizontal aggregation methods

Horizontal aggregation methods operate row-wise across columns and are only available on `DataFrame`:

- `.max_horizontal()`
- `.min_horizontal()`
- `.mean_horizontal(...)`
- `.sum_horizontal()`

Explain that these require access to the row as a whole and cannot be expressed in a `LazyFrame` plan.

## 3. DataFrame-specific aggregation methods

- `.product()` is only available on `DataFrame`.

This row-wise or column-wise product operation depends on concrete data availability.

## 4. Use cases

- Use `LazyFrame` aggregation for analytical grouping and summary queries on large datasets.
- Use `DataFrame` aggregation for exploratory data analysis and when row-wise horizontal reductions are needed.

## 5. GroupBy aggregation

GroupBy aggregation methods are available in both APIs. Use methods like `.agg()`, `.sum()`, `.mean()`, `.max()`, `.min()`, `.n_unique()`, `.quantile(...)`, and `.count()` inside a groupby context.

## 6. Answering style

When describing aggregation, clarify whether the method is vertical or horizontal and whether it is supported by lazy mode. Always show a concise code example for the supported API.
