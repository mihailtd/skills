---
name: python-polars-descriptive-methods
description: Instructs the agent on Polars descriptive methods for DataFrame and LazyFrame, including plan visualization and summary statistics.
---

# Python Polars Descriptive Methods Guidelines

You are an expert on Polars descriptive methods. When asked about summary, debugging, or plan-inspection methods, explain their availability and purpose for `DataFrame` and `LazyFrame`.

## 1. DataFrame descriptive methods

The `DataFrame` API provides many descriptive methods for inspecting data:

- `.approx_n_unique()`
- `.describe(...)`
- `.estimated_size(...)`
- `.glimpse()`
- `.is_duplicated()`
- `.is_empty()`
- `.is_unique()`
- `.n_chunks()`
- `.n_unique(...)`
- `.null_count()`
- `.shape`
- `.width`

Explain that these methods operate on materialized data and are useful for exploratory analysis.

## 2. LazyFrame descriptive methods

`LazyFrame` is limited to plan inspection methods:

- `.explain(...)`
- `.show_graph(...)`
- `.collect_schema()`

Note that `LazyFrame.describe()` exists as an operation but it is executed by collecting the result—it is not a pure plan-only method.

## 3. Use cases

- Use `DataFrame.describe()` for summary statistics on concrete data.
- Use `DataFrame.estimated_size()` to estimate memory footprint.
- Use `LazyFrame.explain()` and `LazyFrame.show_graph()` to understand query optimization and execution order.
- Use `LazyFrame.collect_schema()` when you need the resulting schema without collecting full rows.

## 4. Practical guidance

- When logging or debugging a lazy pipeline, output `lf.explain()` before `collect()`.
- Use `df.glimpse()` to get a quick overview of a materialized DataFrame.
- Use `df.is_empty()` and `df.is_unique()` when validating data shapes.

## 5. Answering style

When asked about descriptive methods, emphasize which information is available before execution (`LazyFrame`) versus after execution (`DataFrame`). Give examples of plan inspection versus data summaries.
