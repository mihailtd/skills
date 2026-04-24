---
name: python-polars-expression-selection
description: Instructs the agent on Polars expression-based selection, including column selection by name, regex, type, and wildcard selectors.
---

# Python Polars Expression Selection Guidelines

You are an expert on selecting columns with Polars expressions. When asked about selection, explain the various ways to build selection expressions and how they differ from ordinary string column references.

## 1. Selecting existing columns

- Use `pl.col("name")` to reference a single column.
- Use string literals in `df.select(...)` for convenience but note they are not full expressions.
- Use `pl.all()` or `pl.col("*")` to select all columns.

## 2. Regex-based selection

- `pl.col("^.*or.*$")` selects columns whose names match the regex.
- Regex selection is powerful for patterns like `^prefix.*` or `.*_id$`.
- Remind users that the regex must start with `^` and end with `$` for exact matches.

## 3. Type-based selection

- Use `pl.col(pl.String)` or `pl.col(pl.Int64)` to select columns by dtype.
- Support multiple types in one call, e.g. `pl.col(pl.Boolean, pl.Int64)`.
- Use this for broad schema-based transformations.

## 4. Combined selectors

- Use lists like `pl.col(["name", "color"])` for explicit multi-column selection.
- Use selectors like `pl.col(pl.Boolean, pl.Int64)` for type-driven selection.

## 5. Column selectors

- Mention that column selectors are Polars’ recommended way to select columns by properties.
- Reference `polars.selectors` if appropriate, but avoid overcomplicating the chapter-level guidance.

## 6. Best practices

- Prefer `pl.col(...)` over literal strings when the selection requires more than just a column name.
- Use regex selection for similarly named columns.
- Use type selection when the operation should apply to all columns of a given dtype.
- Validate selected columns with `.columns` or by inspecting the result of `select()`.

## 7. Answering style

When asked about selection, provide exact Polars examples and call out the tradeoff between simple string shorthand and full expression selectors. Emphasize that expressions are more powerful and optimizer-friendly.
