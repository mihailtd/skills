---
name: python-polars-expression-filtering
description: Instructs the agent on filtering rows with Polars expressions, including comparison operators, boolean logic, and expression composition.
---

# Python Polars Expression Filtering Guidelines

You are an expert on Polars filtering expressions. When asked about row filtering, explain how to build boolean expressions and how to combine them effectively.

## 1. Basic filter syntax

- Use `df.filter(pl.col("x") > 100)` for a simple numeric filter.
- Use `pl.col("name") == "Alice"` for equality filtering.
- Use `df.filter(...)` on both DataFrames and LazyFrames.

## 2. Comparison operators

- `==`, `!=`, `>`, `<`, `>=`, `<=`
- Explain that these return boolean expressions.
- Use `pl.col("str").str.contains(...)` or other namespace methods for more complex conditions.

## 3. Boolean logic

- Combine expressions with `&`, `|`, and `~`.
- Use parentheses to avoid precedence issues: `(cond1) & (cond2)`.
- Explain that `and` / `or` are not valid for expression combination.

## 4. Example patterns

- Filter on multiple conditions: `(pl.col("weight") > 1000) & pl.col("is_round")`
- Use regex selection inside filters when needed.
- Use `pl.col("color").is_in(["orange", "green"])` for membership.

## 5. Best practices

- Prefer expression filters instead of materializing intermediate Series.
- Use lazy filters early in the pipeline for pruning.
- Avoid row-wise Python loops; express conditions declaratively.

## 6. Answering style

When describing filtering, show a pair of expressions and how they combine. Emphasize that the filter condition itself is an expression and that it is evaluated lazily when the plan is collected.
