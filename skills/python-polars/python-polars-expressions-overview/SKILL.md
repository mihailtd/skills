---
name: python-polars-expressions-overview
description: Instructs the agent on Polars expression fundamentals, their lazy semantics, and how expressions are applied across DataFrames and LazyFrames.
---

# Python Polars Expressions Overview Guidelines

You are an expert on Polars expressions. When asked about expressions, explain the concept clearly, distinguish them from regular expressions, and show how they are the foundation of Polars’ API.

## 1. What a Polars expression is

- A Polars expression is a lazy description of a computation that produces one or more Series.
- Expressions are not executed until they are passed to a DataFrame or LazyFrame method.
- They are reusable Python objects and can be applied to multiple frames.

## 2. Expressions versus regular expressions

- Do not confuse Polars expressions with regex patterns.
- Regexes are text-matching patterns used by methods like `pl.col()` and `Expr.str.replace()`.
- Expressions are `pl.Expr` objects that describe data transformations.

## 3. Where expressions are used

- `df.select(...)`
- `df.with_columns(...)`
- `df.filter(...)`
- `df.group_by(...).agg(...)`
- `df.sort(...)`
- `lf.with_columns(...)`, `lf.filter(...)`, and other lazy methods

Always tie example usage to these methods.

## 4. Expression properties

- Lazy: they describe work without doing it.
- Reusable: the same expression object can be used on different frames.
- Expressive: a single expression can compute new columns, filter rows, or create multiple outputs.
- Efficient: Polars can optimize and execute multiple expressions in parallel.

## 5. Practical guidance

- Prefer expressions over direct Series operations for performance and optimizer compatibility.
- Use `pl.col(...)` to reference existing columns and `pl.lit(...)` for literal values.
- Use expression chaining and aliases to keep transformations readable.
- Use `.lazy()` / `.collect()` when switching between eager and lazy APIs.

## 6. Answering style

When responding, lead with the expression concept, show an example that applies the expression, and explain why it is the recommended idiom in Polars. Mention that expressions are the foundation of the `Express` part of the Polars API.
