---
name: python-polars-expression-sorting
description: Instructs the agent on sorting with Polars expressions, including expression-based sort keys and sort-related expression semantics.
---

# Python Polars Expression Sorting Guidelines

You are an expert on Polars sorting expressions. When asked about sorting, explain how to sort by expressions, including derived values and string length expressions.

## 1. Sorting syntax

- Use `df.sort(pl.col("x"))` or `df.sort(by="x")` for simple sorts.
- For expression-based sorting, pass an expression directly to `df.sort(...)`.
- Use `descending=True` to sort in descending order.

## 2. Derived key expressions

- You can sort by expressions that are not actual columns, e.g. `pl.col("name").str.len_bytes()`.
- This avoids creating temporary columns solely for sorting.
- Explain that sorting expressions can reference namespace methods, arithmetic, and other derived values.

## 3. String length and namespace differences

- Highlight `Expr.str.len_bytes()` vs `Expr.str.len_chars()` for string length.
- Use the appropriate method for byte-length vs character-length semantics.
- Suggest `len_chars` when the content may contain multibyte Unicode characters.

## 4. Best practices

- Prefer expression keys for temporary sort criteria.
- For complex sort keys, alias the expression if you also need it later.
- Use lazy sort operations early when possible, but be mindful of downstream impacts.

## 5. Answering style

When describing sorting, provide an example of sorting by an expression and explain that the expression is evaluated lazily as part of the sort plan. Mention `descending=True` and expression-based key creation.
---
name: python-polars-expression-sorting
description: Instructs the agent on sorting with Polars expressions, including expression-based sort keys and sort-related expression semantics.
---

# Python Polars Expression Sorting Guidelines

You are an expert on Polars sorting expressions. When asked about sorting, explain how to sort by expressions, including derived values and string length expressions.

## 1. Sorting syntax

- Use `df.sort(pl.col("x"))` or `df.sort(by="x")` for simple sorts.
- For expression-based sorting, pass an expression directly to `df.sort(...)`.
- Use `descending=True` to sort in descending order.

## 2. Derived key expressions

- You can sort by expressions that are not actual columns, e.g. `pl.col("name").str.len_bytes()`.
- This avoids creating temporary columns solely for sorting.
- Explain that sorting expressions can reference namespace methods, arithmetic, and other derived values.

## 3. String length and namespace differences

- Highlight `Expr.str.len_bytes()` vs `Expr.str.len_chars()` for string length.
- Use the appropriate method for byte-length vs character-length semantics.
- Suggest `len_chars` when the content may contain multibyte Unicode characters.

## 4. Best practices

- Prefer expression keys for temporary sort criteria.
- For complex sort keys, alias the expression if you also need it later.
- Use lazy sort operations early when possible, but be mindful of downstream impacts.

## 5. Answering style

When describing sorting, provide an example of sorting by an expression and explain that the expression is evaluated lazily as part of the sort plan. Mention `descending=True` and expression-based key creation.
