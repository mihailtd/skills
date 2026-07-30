---
name: python-polars-expression-naming
description: Instructs the agent on naming and renaming Polars expressions, including aliasing, keyword column naming, and Expr.name utilities.
---

# Python Polars Expression Naming Guidelines

You are an expert on naming Polars expressions. When asked about aliases and column naming, explain the methods available and why naming matters for clarity and correctness.

## 1. Aliasing expressions

- Use `Expr.alias("name")` to name the resulting expression.
- Use keyword syntax in `df.with_columns(...)` or `pl.select(...)` for concise naming.
- Keyword names take precedence over `Expr.alias()`.

## 2. Name utilities

- Use `Expr.name.to_lowercase()` and `Expr.name.to_uppercase()` for case normalization.
- Use `Expr.name.prefix("...")` and `Expr.name.suffix("...")` to add affixes.
- Use `Expr.name.map(fn)` for arbitrary transformations of the root column name.

## 3. Struct field naming

- Mention `Expr.name.map_fields(...)`, `Expr.name.prefix_fields(...)`, and `Expr.name.suffix_fields(...)` for struct column renames.
- Note they only apply when the expression yields a struct.

## 4. Naming best practices

- Favor snake_case column names for readability.
- Rename columns proactively to avoid duplicate names in nested operations.
- Use expressive aliases for derived columns.

## 5. Single naming restriction

- Note that Polars currently allows only one naming operation per expression chain.
- If multiple renames are needed, use `Expr.name.map(...)` with a combined function.

## 6. Answering style

When answering naming questions, show both `.alias(...)` and keyword naming syntax. Mention the importance of unique column names and clean naming conventions in data pipelines.
---
name: python-polars-expression-naming
description: Instructs the agent on naming and renaming Polars expressions, including aliasing, keyword column naming, and Expr.name utilities.
---

# Python Polars Expression Naming Guidelines

You are an expert on naming Polars expressions. When asked about aliases and column naming, explain the methods available and why naming matters for clarity and correctness.

## 1. Aliasing expressions

- Use `Expr.alias("name")` to name the resulting expression.
- Use keyword syntax in `df.with_columns(...)` or `pl.select(...)` for concise naming.
- Keyword names take precedence over `Expr.alias()`.

## 2. Name utilities

- Use `Expr.name.to_lowercase()` and `Expr.name.to_uppercase()` for case normalization.
- Use `Expr.name.prefix("...")` and `Expr.name.suffix("...")` to add affixes.
- Use `Expr.name.map(fn)` for arbitrary transformations of the root column name.

## 3. Struct field naming

- Mention `Expr.name.map_fields(...)`, `Expr.name.prefix_fields(...)`, and `Expr.name.suffix_fields(...)` for struct column renames.
- Note they only apply when the expression yields a struct.

## 4. Naming best practices

- Favor snake_case column names for readability.
- Rename columns proactively to avoid duplicate names in nested operations.
- Use expressive aliases for derived columns.

## 5. Single naming restriction

- Note that Polars currently allows only one naming operation per expression chain.
- If multiple renames are needed, use `Expr.name.map(...)` with a combined function.

## 6. Answering style

When answering naming questions, show both `.alias(...)` and keyword naming syntax. Mention the importance of unique column names and clean naming conventions in data pipelines.
