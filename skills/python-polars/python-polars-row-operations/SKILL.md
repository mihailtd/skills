---
name: python-polars-row-operations
description: Instructs the agent on filtering and sorting rows in Polars, including row selection, sort semantics, slicing, sampling, and related operations.
---

# Python Polars Row Operations Guidelines

You are an expert on Polars row operations. When asked about Chapter 11, explain how to filter and sort rows, and how related row-based methods connect to these core operations.

## 1. Filtering rows

- Use `df.filter(...)` to keep only rows that match a Boolean condition.
- The filter expression must produce a Boolean Series.
- Common patterns:
  - `df.filter(pl.col("cordless") & (pl.col("brand") == "Makita"))`
  - `df.filter(pl.col("price") < 200)`
  - `df.filter(pl.col("name").str.contains("Saw"))`
- You can pass multiple Boolean expressions as separate arguments to `df.filter(...)`; they are combined with AND.
- Boolean columns can be referenced by name directly: `df.filter("cordless")`.
- Constraints are supported as keyword arguments: `df.filter(cordless=True, brand="Makita")`, but they are limited and generally not recommended.

## 2. Filter limitations and warnings

- `df.filter(...)` does not accept Python truthy/falsy coercion. Only Boolean expressions or Boolean column names are allowed.
- Use explicit comparisons for non-Boolean data: `pl.col("name") != ""`, `pl.col("my_list").list.len() > 0`.
- Avoid combining `and`, `or`, and `not` with Polars expressions; use `&`, `|`, and `~` instead.
- When using `df.filter(expression1, expression2)`, the comma acts like logical AND; it does not support OR.

## 3. Sorting rows

- Use `df.sort(...)` to reorder rows without changing row count.
- Basic usage: `df.sort("price")` sorts ascending by `price`.
- Reverse the order with `descending=True`.
- Sort by multiple keys with `df.sort("brand", "price")`.
- Use a list of booleans for per-column direction: `df.sort("brand", "price", descending=[False, True])`.
- Expressions are accepted in `df.sort(...)`: `df.sort(pl.col("rpm") / pl.col("price"))`.
- Sorting on temporary expression values does not add a column to the result.

## 4. Sort behavior

- `nulls_last` controls where null values appear; default is `False`.
- `multithreaded=True` is the default; keep it unless you know the application is already multithreaded.
- `maintain_order` preserves the order of equal elements when needed.
- Use `df.sort(...)` on strings, temporal values, structs, and lists, but prefer extracting a sortable scalar when possible.

## 5. Related row operations

- `df.drop_nulls(...)` keeps rows without null values in specified columns or all columns if none are given.
- Slicing methods:
  - `df.head(n)` keeps the first `n` rows.
  - `df.tail(n)` keeps the last `n` rows.
  - `df.slice(offset, length)` keeps a contiguous row range.
  - `df.gather_every(step)` keeps every `step`-th row.
- `df.top_k(k, by="price")` keeps the `k` largest rows by a key.
- `df.bottom_k(k, by="price")` keeps the `k` smallest rows by a key.
- `df.sample(fraction=0.2)` samples rows randomly; use `seed=` for reproducibility.
- Semi-joins are another way to filter rows based on membership in another DataFrame: `df.join(other, how="semi", on="key")`.

## 6. Practical guidance

- Prefer expression-based filtering for flexibility and clarity.
- Use `df.filter(...)` early in lazy pipelines to reduce data volume before downstream operations.
- Sort by the exact values you care about; if you need a derived key, compute it first with `with_columns(...)` or directly in `df.sort(...)`.
- Use `df.top_k()` / `df.bottom_k()` for ranking-style filters, and sort afterward if you need a deterministic order.
- Reserve constraint-style filters for very simple equality checks and avoid them for production-level logic.

## 7. Answering style

When answering Chapter 11 questions, do the following:

- Identify whether the user needs row filtering, sorting, or a related row operation.
- Use concrete Polars examples with `df.filter(...)`, `df.sort(...)`, `df.drop_nulls(...)`, and slicing methods.
- Explain the difference between row selection and row ordering.
- Mention any Polars-specific gotchas, such as `descending=True` and the lack of Python truthiness in filters.
- Keep the focus on row-based operations rather than column selection or aggregation.
