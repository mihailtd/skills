---
name: python-polars-groupby-methods
description: Instructs the agent on Polars GroupBy methods for DataFrame and LazyFrame, including aggregation and iteration semantics.
---

# Python Polars GroupBy Methods Guidelines

You are an expert on Polars GroupBy. When asked about grouped aggregation, describe how GroupBy behaves in both `DataFrame` and `LazyFrame` contexts.

## 1. Shared groupby methods

The following methods are available in both `DataFrame` and `LazyFrame` GroupBy contexts:

- `.agg(...)`
- `.all()`
- `.count()`
- `.first()`
- `.head(...)`
- `.last()`
- `.len()`
- `.map_groups(...)`
- `.max()`
- `.mean()`
- `.median()`
- `.min()`
- `.n_unique()`
- `.quantile(...)`
- `.sum()`
- `.tail(...)`

Explain that these operations are fully supported in lazy plans and eager materialized groupby computations.

## 2. DataFrame-only GroupBy behavior

- `DataFrame.groupby(...).__iter__()` is available only for eager `DataFrame`.
- Iterating groups requires materialized data.

## 3. Use cases

- Use `LazyFrame.group_by(...).agg(...)` for scalable grouped analytics.
- Use `DataFrame.group_by(...).__iter__()` for manual per-group inspection or custom Python loops.

## 4. Practical guidance

- Prefer `agg()` in lazy pipelines because it composes naturally into the plan.
- Use `.map_groups(...)` when you need to run a custom function per group and are prepared to materialize the group data.
- Avoid group iteration on large datasets unless the grouping cardinality is small.

## 5. Answering style

When describing GroupBy methods, note the symmetry between eager and lazy APIs for aggregation, and the key difference that direct iteration is only available once the data is materialized.
