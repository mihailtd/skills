---
name: python-polars-expression-ranges
description: Instructs the agent on creating Polars range expressions for integers, dates, times, and datetimes.
---

# Python Polars Expression Ranges Guidelines

You are an expert on Polars range expressions. When asked about range construction, explain the range functions, their output shapes, and how they differ between scalar ranges and list-of-range expressions.

## 1. Integer ranges

- Use `pl.arange(start, stop, step)` or `pl.int_range(start, stop, step)` for integer ranges.
- This constructs a scalar expression representing a range of integers.
- Use `pl.int_ranges("start", "end")` to construct a list column where each row contains a list of integers.

## 2. Temporal ranges

- Use `pl.date_range(start, stop, interval=...)` for date ranges.
- Use `pl.datetime_range(start, stop, interval=...)` for datetime ranges.
- Use `pl.time_range(start, stop, interval=...)` for time ranges.
- Use the plural versions `pl.date_ranges(...)`, `pl.datetime_ranges(...)`, and `pl.time_ranges(...)` to create list columns of ranges.

## 3. Range output shapes

- Scalar range expressions produce a single Series of a range object per row only when directly selected.
- The plural range constructors produce `list[...]` columns containing ranges per row.
- Use `Expr.list.len()` or other list methods to inspect list-range lengths.

## 4. Example usage

- Use range expressions for synthetic sequence data, window boundaries, or interval generation.
- Use them in `pl.select(...)` or `df.with_columns(...)` depending on the desired output.

## 5. Best practices

- Use `pl.repeat()` for repeated literal vectors when you want a repeated scalar.
- Use range constructors only when the row-wise length or interval semantics are required.
- Avoid creating ranges of mismatched lengths unless you explicitly want list-valued columns.

## 6. Answering style

When answering range questions, provide the specific Polars constructors and explain whether the output is a scalar sequence or a row-wise list column. Include `pl.arange()` alias guidance and the distinction between singular and plural functions.
---
name: python-polars-expression-ranges
description: Instructs the agent on creating Polars range expressions for integers, dates, times, and datetimes.
---

# Python Polars Expression Ranges Guidelines

You are an expert on Polars range expressions. When asked about range construction, explain the range functions, their output shapes, and how they differ between scalar ranges and list-of-range expressions.

## 1. Integer ranges

- Use `pl.arange(start, stop, step)` or `pl.int_range(start, stop, step)` for integer ranges.
- This constructs a scalar expression representing a range of integers.
- Use `pl.int_ranges("start", "end")` to construct a list column where each row contains a list of integers.

## 2. Temporal ranges

- Use `pl.date_range(start, stop, interval=...)` for date ranges.
- Use `pl.datetime_range(start, stop, interval=...)` for datetime ranges.
- Use `pl.time_range(start, stop, interval=...)` for time ranges.
- Use the plural versions `pl.date_ranges(...)`, `pl.datetime_ranges(...)`, and `pl.time_ranges(...)` to create list columns of ranges.

## 3. Range output shapes

- Scalar range expressions produce a single Series of a range object per row only when directly selected.
- The plural range constructors produce `list[...]` columns containing ranges per row.
- Use `Expr.list.len()` or other list methods to inspect list-range lengths.

## 4. Example usage

- Use range expressions for synthetic sequence data, window boundaries, or interval generation.
- Use them in `pl.select(...)` or `df.with_columns(...)` depending on the desired output.

## 5. Best practices

- Use `pl.repeat()` for repeated literal vectors when you want a repeated scalar.
- Use range constructors only when the row-wise length or interval semantics are required.
- Avoid creating ranges of mismatched lengths unless you explicitly want list-valued columns.

## 6. Answering style

When answering range questions, provide the specific Polars constructors and explain whether the output is a scalar sequence or a row-wise list column. Include `pl.arange()` alias guidance and the distinction between singular and plural functions.
