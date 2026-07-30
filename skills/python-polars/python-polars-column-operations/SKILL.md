---
name: python-polars-column-operations
description: Instructs the agent on selecting and creating columns in Polars, including selectors, df.select(), df.with_columns(), and related column operations.
---

# Python Polars Column Operations Guidelines

You are an expert on Polars column operations. When asked about Chapter 10, explain how to select existing columns, use selectors, create new columns, and perform related operations such as dropping, renaming, stacking, and adding row indices.

## 1. Selecting columns

- Use `df.select(...)` to choose which columns to keep.
- String names are convenient for simple selections, e.g. `df.select("name", "mass")`.
- Use `pl.col("name")` or `pl.col("^pattern$")` when you need an expression-based selection.
- The selected columns appear in the order you specify them.
- You can select the same column more than once only if you alias one of the duplicates.

## 2. Column selectors

- Import selectors with `import polars.selectors as cs`.
- Selectors are expressions under the hood and can be used anywhere column selection is allowed, including `df.select()`, `df.filter()`, `df.group_by()`, and `df.with_columns()`.
- Use name-based selectors like `cs.by_name(...)`, `cs.starts_with(...)`, `cs.ends_with(...)`, `cs.contains(...)`, `cs.matches(...)`, `cs.alpha()`, `cs.digit()`, and `cs.alphanumeric()`.
- Use type-based selectors like `cs.numeric()`, `cs.string()`, `cs.boolean()`, `cs.temporal()`, `cs.integer()`, `cs.float()`, `cs.by_dtype(pl.List(pl.String))`, and other `cs.*` functions.
- Use position-based selectors like `cs.by_index(...)`, `cs.first()`, and `cs.last()` when you need column position rather than name or type.
- Explain that `cs.by_index(range(start, stop, step))` can select every nth column and that negative indices can select from the end.

## 3. Combining selectors

- Selectors support set-like operators: `|`, `&`, `-`, `^`, and `~`.
- `x | y` selects columns in either selector.
- `x & y` selects only columns common to both selectors.
- `x - y` selects columns in `x` but not in `y`.
- `x ^ y` selects columns in exactly one selector.
- `~x` selects all columns not in `x`.
- Point out that these operators preserve column order and can produce an empty selection.

## 4. Creating new columns

- Use `df.with_columns(...)` to add one or more new columns to the right end of the DataFrame.
- Each new column is expressed as `alias=expr` or `expr.alias("name")`.
- Creating a new column with an existing column name overwrites the original column.
- To keep the original column, choose a new alias name.
- `df.with_columns(...)` is syntactic sugar for `df.select(pl.all(), <new_columns>)`.
- If you want a new column to appear in the middle, use `.alias(...)` inside `select()` rather than keyword syntax.

## 5. Expression dependency rules

- New columns created in one `with_columns()` call cannot depend on other columns being created in the same call.
- If one new column depends on another, use multiple `with_columns()` calls in sequence.
- Example: first create `bmi`, then in a separate call create `bmi_cat` from `pl.col("bmi")`.

## 6. Related column operations

- Dropping columns: use `df.drop("col1", "col2", strict=False)` to remove columns without raising if they are missing.
- Renaming columns: use `df.rename({"old": "new"})` or `df.rename(lambda s: ...)` for function-based renames.
- Stacking horizontally: use `df.hstack(...)` to combine another DataFrame or Series with the same row count.
- Adding row indices: use `df.with_row_index(name="id", offset=1)` to add an increasing integer column.
- Explain that stacking requires proper alignment on row order and is distinct from joins.

## 7. Best practices

- Prefer explicit `pl.col(...)` or selector usage for complex selections.
- Use selectors when you need to choose columns by pattern, type, or position.
- Avoid overusing position-based selectors unless the column order is intentionally meaningful.
- Use `df.drop(...)` when it is easier to specify unwanted columns than wanted columns.
- Use `df.rename(...)` for schema transformations rather than manual `select(...)` renames.
- Keep column creation chains readable with aliases and clear expressions.

## 8. Answering style

When answering Chapter 10 questions, do the following:

- Identify whether the user wants to select existing columns, create new columns, or perform a related DataFrame column operation.
- Recommend the simplest selection mechanism that meets the need: literal names for small fixed sets, `pl.col(...)` for expression-based selection, selectors for flexible pattern/type/position selection.
- Explain any Polars-specific behavior such as parallel evaluation of new columns in `with_columns()` and overwrite semantics.
- If the question is about selector composition, show one example using set operators and explain the result.
- Keep the focus on column-oriented operations rather than row filtering or aggregation.
