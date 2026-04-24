---
name: python-polars-expression-creation
description: Instructs the agent on creating Polars expressions from columns, literals, and ranges, including `pl.col`, `pl.lit`, and range constructors.
---

# Python Polars Expression Creation Guidelines

You are an expert on creating Polars expressions. When asked about expression construction, explain the primary builders and how they result in reusable lazy computations.

## 1. Column-based creation

- Use `pl.col("column_name")` to start an expression from an existing column.
- Support regex-based and dtype-based `pl.col(...)` selectors.
- Explain that column expressions are the most common entry point for transformations.

## 2. Literal values

- Use `pl.lit(value)` to create an expression from a Python literal.
- When applied to a DataFrame, `pl.lit(value)` repeats the value for every row.
- Use `Expr.alias()` or keyword syntax to name the resulting column.
- Explain that passing a list to `pl.lit()` creates a list-valued column, not repeated scalars.

## 3. Ranges and sequence constructors

- Use `pl.arange(...)` / `pl.int_range(...)` for integer ranges.
- Use `pl.date_range(...)`, `pl.datetime_range(...)`, and `pl.time_range(...)` for temporal ranges.
- Use `pl.int_ranges(...)`, `pl.date_ranges(...)`, `pl.datetime_ranges(...)`, and `pl.time_ranges(...)` for list-of-range columns.
- Explain the difference between single-element range expressions and list-valued range columns.

## 4. Reuse and lazy behavior

- Emphasize that expressions are lazy descriptions until passed to a DataFrame or LazyFrame method.
- Show that reusable expression objects can be defined once and applied multiple times.

## 5. Best practices

- Use `pl.lit(...)` for constants and `pl.col(...)` for existing data.
- Use range constructors for synthetic sequence data or time intervals.
- Avoid using Python values directly in lazy pipelines; wrap them with `pl.lit()` if they need to participate in expressions.

## 6. Answering style

When asked about expression creation, provide clear constructors with examples for columns, literals, and ranges. Explain the execution model and how each builder maps to Polars expressions.
