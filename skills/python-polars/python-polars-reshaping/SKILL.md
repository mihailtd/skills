---
name: python-polars-reshaping
description: Instructs the agent on Polars reshaping methods, including pivot, unpivot, transpose, explode, and partition_by.
---

# Python Polars Reshaping Guidelines

You are an expert on Polars reshaping. When asked about changing DataFrame shape for analysis, visualization, or downstream aggregation, explain the right operation for wide/long conversion, nested list expansion, and dataset partitioning.

## 1. Reshaping concepts

- Explain the difference between wide and long data:
  - wide: many columns, fewer rows, variables stored as separate columns.
  - long: fewer columns, more rows, variables stored as values in a single column.
- Mention tidy data rules:
  - each variable gets its own column,
  - each observation gets its own row,
  - each value gets its own cell.
- Use reshaping when the current layout is not aligned with aggregation, filtering, or visualization goals.
- Distinguish reshaping from joining and concatenation: reshaping rearranges a single dataset, it does not combine rows from different sources.

## 2. Pivoting to a wider layout

- `df.pivot(...)` is the canonical operation for long-to-wide transformation.
- Common arguments:
  - `index`: row identifier columns.
  - `on`: column whose values become new columns.
  - `values`: column whose values fill the pivoted cells.
  - `aggregate_function`: required when duplicates exist for a single pivot cell.
  - `maintain_order`, `sort_columns`, `separator` for predictable output.
- If asked about duplicate values, recommend `aggregate_function="mean"`, `"sum"`, `"min"`, `"max"`, `"median"`, `"len"`, or expression-based aggregation like `pl.element().max() - pl.element().min()`.
- Note the lazy pivot limitation: `df.pivot()` needs column values known ahead of time. For `LazyFrame`, explain the workaround with `group_by(...).agg(...)` and explicit unique column values when necessary.

## 3. Unpivoting to a longer layout

- `df.unpivot(...)` converts wide rows into long rows.
- Common arguments:
  - `index` / `id_vars`: columns to keep as identifiers.
  - `on` / `value_vars`: columns to melt into rows.
  - `variable_name`: name of the resulting column holding the original column names.
  - `value_name`: name of the resulting column holding the values.
- Emphasize that columns not in `index` or `on` are dropped unless intentionally included.
- Use `unpivot` when many measurements are stored in separate columns and you want a tidy table with one measure per row.

## 4. Transposing a DataFrame

- `df.transpose(...)` flips columns and rows diagonally.
- This is a DataFrame-only operation.
- Typical use cases:
  - converting a small metadata table into a row-based view,
  - moving column headers into a row when the shape is already fixed.
- Key arguments:
  - `include_header`: whether the original column names become a new column.
  - `header_name`: name for the column holding the original column headers.
  - `column_names`: explicit names for the transposed output columns.
- Warn that transposing usually changes semantics and should be used when the data is truly matrix-like or when all rows share the same type constraints.

## 5. Exploding nested values

- `df.explode(...)` expands list/array columns into separate rows.
- The method is DataFrame-only and works one layer at a time for nested lists.
- If multiple columns are exploded together, they must produce the same number of elements per row.
- Explain common patterns:
  - exploding a single list column to normalize nested observations,
  - exploding nested structures with repeated calls to flatten one level at a time.
- Mention `strict=False` only when constructing nested data that may not be uniform, but keep the focus on deterministic exploding behavior.

## 6. Partitioning into grouped DataFrames

- `df.partition_by(...)` splits one DataFrame into multiple DataFrames by group.
- Useful when you need separate materialized subsets, for example to feed different downstream operations or to export group-specific files.
- Useful arguments:
  - `by` / `*more_by`: grouping columns.
  - `maintain_order`: keep group order deterministic.
  - `include_key`: include or exclude the grouping key column in each partition.
  - `as_dict`: return a dictionary keyed by the group values.
- Point out that `partition_by` is different from `group_by(...)` because it returns separate DataFrames rather than a grouped aggregation object.

## 7. Practical guidance

- Use `pivot` when you want a compact table with one column per category.
- Use `unpivot` when you need tidy data with one observation per row.
- Use `transpose` for matrix-style, metadata, or small summary tables, not for large analytic tables.
- Use `explode` to normalize nested list/array columns into row-oriented form.
- Use `partition_by` when you want materialized subsets rather than a single combined table.
- When answering, relate the method choice back to the intended analysis: aggregation, visualization, or row-level unpacking.

## 8. Answering style

- Be explicit about whether the operation is eager-only or if it can be expressed lazily.
- Mention the effect on row and column counts: `pivot` widens, `unpivot` lengthens, `transpose` swaps axes, `explode` multiplies rows, `partition_by` creates multiple tables.
- Prefer concrete Polars examples that use `pl.DataFrame(...)` and the named arguments from the chapter.
- Avoid recommending join/concat operations here unless the user explicitly asks about combining separate datasets; this skill is about reshaping a single dataset.
