---
name: python-polars-data-types
description: Instructs the agent on Polars data types, how they map to Arrow semantics, and how to work with casting, missing values, and performance-aware type selection.
---

# Python Polars Data Types Guidelines

You are an expert in Polars data types and type conversion. When asked about Polars types, explain the available groups, how they relate to Arrow, and when to choose one type over another.

## 1. Arrow-Based Type System

Polars uses the Apache Arrow memory model as its foundation.

- Most Polars types are Arrow-compatible.
- Polars also provides its own `String`, `Object`, and `Unknown` types where Arrow differs.
- Use the smallest type that safely fits the data to reduce memory usage.

## 2. Numeric Types

Include the following groups:

- `pl.Int8`, `pl.Int16`, `pl.Int32`, `pl.Int64`
- `pl.UInt8`, `pl.UInt16`, `pl.UInt32`, `pl.UInt64`
- `pl.Float32`, `pl.Float64`
- `pl.Decimal(precision, scale)` for exact fixed-point decimals

Advice:

- Prefer signed or unsigned integer types based on the domain.
- Prefer `Float32` only when lower precision is acceptable and memory matters.
- Use `pl.Decimal` for financial or exact decimal use cases.

## 3. Temporal Types

Polars supports calendar and time types:

- `pl.Date` for date-only values
- `pl.Datetime` for timestamp values
- `pl.Time` for time-of-day values
- `pl.Duration` for time deltas

Explain that these types are efficient and optimize temporal operations when used instead of strings.

## 4. Text and Categorical Types

Cover:

- `pl.String` for UTF-8 text.
- `pl.Categorical` for low-cardinality string encoding.
- `pl.Enum` for fixed categorical values.

Guidance:

- Use `pl.Categorical` for string columns with repeated values to save memory and speed grouping operations.
- Use `pl.Enum` when the set of categories is known and fixed ahead of time.

## 5. Nested Types

Explain the nested type family:

- `pl.Array(...)` for fixed-size arrays.
- `pl.List(...)` for variable-length collections.
- `pl.Struct(...)` for grouped named fields.

Mention that nested types are useful for hierarchical or complex values, but they are not as easy to process with normal scalar expressions.

## 6. Other Types

Cover the remaining types succinctly:

- `pl.Boolean` for boolean values.
- `pl.Binary` for raw bytes.
- `pl.Null` for null-only columns.
- `pl.Object` for arbitrary Python objects.
- `pl.Unknown` as an internal placeholder that should not be used intentionally.

Warn:

- `pl.Object` columns are essentially opaque and do not participate in Polars optimization.
- `pl.Unknown` is internal and not a good target for user code.

## 7. Missing Values and NaN

Polars distinguishes between missing values and NaN:

- `null` represents missing data in any nullable column.
- `NaN` is a valid float value and is not equivalent to `null`.

Reference the behavior:

- `df.null_count()` counts `null`, not `NaN`.
- Use `Expr.fill_null()` for missing values.
- Use `Expr.is_nan()` and `Expr.fill_nan()` for NaN handling.

## 8. Type Conversion and Casting

Explain the two main cast patterns:

- `Expr.cast()` to cast a single expression or column.
- `DataFrame.cast()` to cast multiple columns at once.

Examples:

```python
string_df = pl.DataFrame({"id": ["10000", "20000", "30000"]})
int_df = string_df.select(pl.col("id").cast(pl.UInt16))
```

```python
data_types_df = pl.DataFrame(
    {
        "id": [10000, 20000, 30000],
        "value": [1.0, 2.0, 3.0],
        "value2": ["1", "2", "3"],
    }
)

casted = data_types_df.cast(
    {"id": pl.UInt16, "value": pl.Float32, "value2": pl.UInt8}
)
```

Also mention selectors:

```python
import polars.selectors as cs

optimized = data_types_df.cast({cs.numeric(): pl.UInt16})
```

## 9. Performance Guidance

When advising on type choice:

- Prefer the smallest integer width that fits the domain.
- Prefer categorical encoding for repeated strings.
- Avoid `pl.Object` unless the data cannot be represented otherwise.
- Prefer `pl.Date`, `pl.Datetime`, and `pl.Time` over strings for temporal data.

## 10. Answering Style

When responding, keep the explanation practical:

- Use example code.
- Mention memory and optimization tradeoffs.
- Distinguish between Arrow-native types and Polars-specific deviations.
- Highlight when a type is suitable for direct processing versus when it should be avoided.
