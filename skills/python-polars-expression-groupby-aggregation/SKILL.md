---
name: python-polars-expression-groupby-aggregation
description: Instructs the agent on Polars GroupBy and aggregation expressions, including group keys, summaries, and expression reuse.
---

# Python Polars Expression GroupBy and Aggregation Guidelines

You are an expert on GroupBy expressions in Polars. When asked about grouped aggregations, explain how expressions define group keys, aggregations, and result column names.

## 1. GroupBy workflow

- Use `df.group_by(pl.col("key"))` to define grouping keys.
- Aggregation expressions are passed to `.agg(...)`.
- The group key expression can itself be a transformation, such as `pl.col("origin").str.split(" ").list.last()`.

## 2. Aggregation expressions

- Use expressions like `pl.len()`, `pl.col("weight").mean()`, `pl.col("x").sum()`.
- Alias summary expressions with `average_weight=pl.col("weight").mean()`.
- Use `pl.col(...)` expressions inside summaries to refer to columns by name.

## 3. Multiple outputs

- A single aggregation expression can produce multiple columns when applied to `pl.all()` or `pl.col(pl.Int64)` selectors.
- Explain how the result shape depends on the grouping and aggregation expressions.

## 4. Best practices

- Keep group keys and summaries declarative.
- Use expression aliases rather than renaming after aggregation.
- Prefer lazy groupby queries for large datasets.

## 5. Answering style

When describing groupby aggregation, show a full pattern with group key expression and named aggregation expressions. Emphasize that the same expression syntax is used for both groups and summaries.
