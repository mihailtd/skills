---
name: python-polars-expression-operations
description: Instructs the agent on continuing a Polars expression with additional methods, including element-wise transforms, missing-value handling, smoothing, selection, and summaries.
---

# Python Polars Expression Operations Guidelines

You are an expert on continuing Polars expressions. When asked about Chapter 8 operations, explain how to extend a base expression with additional methods, and classify the result by whether it preserves length, summarizes, or extends the Series.

## 1. Continuing an expression

- A continuing expression starts from a base `pl.Expr` such as `pl.col("x")` or `pl.lit(1)`.
- Additional methods like `.sqrt()`, `.fill_null(0)`, `.rolling_mean(...)`, or `.mean()` extend the expression.
- The continued expression is still lazy until it is passed to a frame method such as `select`, `with_columns`, `filter`, or `agg`.

## 2. Output categories

- Element-wise operations preserve the Series length and compute each value independently.
- Series-wise nonreducing operations also preserve length but depend on the whole Series or neighboring rows.
- Summarizing operations reduce the Series to one value or one row, often used in aggregation contexts.
- Some operations extend the Series length, such as `explode()` or `extend_constant(...)`.

## 3. Key Chapter 8 categories

- Mathematical transforms: `Expr.abs()`, `Expr.exp()`, `Expr.log()`, `Expr.sqrt()`, `Expr.sign()`.
- Trigonometry: `Expr.sin()`, `Expr.cos()`, `Expr.degrees()`, `Expr.radians()`.
- Rounding and binning: `Expr.round()`, `Expr.ceil()`, `Expr.floor()`, `Expr.cut()`, `Expr.qcut()`.
- Missing/infinite values: `Expr.fill_nan()`, `Expr.fill_null()`, `Expr.is_null()`, `Expr.is_nan()`, `Expr.is_infinite()`.
- Nonreducing Series-wise operations: `Expr.cum_sum()`, `Expr.shift()`, `Expr.forward_fill()`, `Expr.is_unique()`, `Expr.sort()`, `Expr.rank()`.
- Rolling/smoothing: `Expr.rolling_mean()`, `Expr.ewm_mean()`, `Expr.rolling_std()`, `Expr.rolling_max()`.
- Summaries: `Expr.mean()`, `Expr.sum()`, `Expr.min()`, `Expr.max()`, `Expr.all()`, `Expr.any()`, `Expr.count()`, `Expr.null_count()`.
- Selecting and reshaping: `Expr.head()`, `Expr.slice()`, `Expr.gather()`, `Expr.unique()`, `Expr.value_counts()`.
- Extending: `Expr.explode()`, `Expr.extend_constant(...)`.

## 4. Practical guidance

- Explain the method category first, then show a concise example.
- For summarizing expressions, note that the value is usually repeated if the original row shape is retained unless used inside `agg()`.
- Use `.alias(...)` or keyword naming to name the result of continued expressions.
- Mention that `Expr.sort()` and `Expr.sort_by()` sort a single column expression, while `df.sort(...)` sorts rows.
- For missing values, distinguish between `NaN` and `null`, and show combined handling with `.fill_nan(...).fill_null(...)` when needed.
- Prefer expression chaining to keep Polars code readable and optimizer-friendly.

## 5. Answering style

When answering questions about continuing expressions, do the following:

- State that the question is about extending a `pl.Expr` chain.
- Identify whether the method is element-wise, Series-wise nonreducing, summarizing, or extending.
- Show a real Polars expression example that uses the method in context.
- Mention that full method coverage is in the Polars API reference, and Chapter 8 focuses on the operation categories rather than every method.
- If the user asks about related methods that fall in other chapters, explain the boundary: Chapter 8 is about continuing expressions, Chapter 9 is about combining expressions, and Chapter 11 is about sorting entire DataFrames.
