---
name: python-polars-manipulation-methods
description: Instructs the agent on Polars DataFrame and LazyFrame manipulation methods, including selection, filtering, joins, reshaping, and transformations.
---

# Python Polars Manipulation Methods Guidelines

You are an expert on Polars manipulation methods. When asked about how to transform or manipulate data, explain the method categories that apply to both `DataFrame` and `LazyFrame`, and note any methods that are DataFrame-only.

## 1. Core selection and transformation

These methods are available for both `DataFrame` and `LazyFrame`:

- `.select(...)`
- `.filter(...)`
- `.with_columns(...)`
- `.with_columns_seq(...)`
- `.with_row_count(...)`
- `.with_row_index(...)`
- `.sort(...)`
- `.head(...)`
- `.tail(...)`
- `.limit(...)`
- `.slice(...)`
- `.drop(...)`
- `.drop_nulls(...)`
- `.fill_null(...)`
- `.fill_nan(...)`
- `.cast(...)`
- `.rename(...)`
- `.replace_column(...)`
- `.set_sorted(...)`
- `.unique(...)`
- `.reverse()`
- `.group_by(...)`
- `.group_by_dynamic(...)`
- `.join(...)`
- `.join_asof(...)`
- `.join_where(...)`
- `.merge_sorted(...)`
- `.partition_by(...)`
- `.melt(...)`
- `.pivot(...)`
- `.unpivot(...)`
- `.unnest(...)`
- `.explode(...)`
- `.slice(...)`
- `.sample(...)`
- `.top_k(...)`
- `.bottom_k(...)`
- `.shift(...)`
- `.rolling(...)`
- `.interpolate()`
- `.to_dummies()`
- `.to_series()`

Explain that these methods are the core building blocks for both eager and lazy workflows.

## 2. DataFrame-only manipulation methods

The following methods require a materialized `DataFrame` and are unavailable on `LazyFrame`:

- `.drop_in_place(...)`
- `.extend(...)`
- `.get_column(...)`
- `.get_column_index(...)`
- `.get_columns()`
- `.hstack(...)`
- `.insert_column(...)`
- `.inspect(...)`
- `.iter_columns()`
- `.iter_rows()`
- `.iter_slices(...)`
- `.join_where(...)` (if the implementation depends on in-memory evaluation)
- `.last()`
- `.merge_sorted(...)` (DataFrame-only behavior may differ)
- `.partition_by()`
- `.pipe(...)` (DataFrame has immediate pipelining semantics)
- `.rechunk()`
- `.row(...)`
- `.rows(...)`
- `.rows_by_key(...)`
- `.set_sorted(...)` when it materializes sort-related flags
- `.shrink_to_fit()`
- `.transpose(...)`
- `.vstack(...)`
- `.hstack(...)`

Explain that DataFrame-only methods often require the actual data layout or row iteration.

## 3. Join and reshape operations

- Joins can be performed in both APIs, but both inputs must be the same type.
- `DataFrame.join(...)` and `LazyFrame.join(...)` are both supported.
- For a `DataFrame` and `LazyFrame` join, convert the eager side with `df.lazy()` or the lazy side with `lf.collect()`.

Reshaping methods such as `.melt(...)`, `.pivot(...)`, `.unpivot(...)`, and `.unnest(...)` are available in both APIs and compose cleanly in lazy plans.

## 4. Practical guidance

- Use `.filter(...)`, `.select(...)`, and `.with_columns(...)` as the primary lazy transformations.
- Use `.sort(...)`, `.group_by(...)`, `.join(...)`, and `.pivot(...)` for analytic workloads.
- Avoid `.iter_rows()` and other row-level iteration methods on large DataFrames.
- Prefer expression-based transformations in lazy mode rather than Python loops.

## 5. Answering style

When asked about manipulation, make it clear which methods are API-agnostic and which require a materialized `DataFrame`. Use examples that show both eager and lazy forms of the same transformation when possible.
