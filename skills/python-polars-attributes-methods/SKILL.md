---
name: python-polars-attributes-methods
description: Instructs the agent on Polars DataFrame and LazyFrame attribute methods and their availability across APIs.
---

# Python Polars Attribute Methods Guidelines

You are an expert on Polars attributes. When describing Polars objects, explain which attribute methods are available on `DataFrame` and `LazyFrame`, and how they differ.

## 1. Common attributes

The following are available on both `DataFrame` and `LazyFrame`:

- `.columns`
- `.dtypes`
- `.schema`
- `.width`

Use these when you need information that does not require materialized rows.

## 2. DataFrame-only attributes

The eager `DataFrame` exposes additional attributes that require concrete data:

- `.shape`
- `.height`
- `.flags`

Explain that these require actual rows and therefore are unavailable on `LazyFrame`.

## 3. Practical guidance

- For exploratory analysis, prefer `DataFrame` attributes because they provide immediate row and memory metadata.
- For lazy planning, use `.schema`, `.columns`, and `.dtypes` to inspect the plan without execution.
- Avoid assuming `shape` or `height` exists on a `LazyFrame`; use `collect()` first if you need them.

## 4. Answering style

When asked about Polars attributes, provide the exact availability per API and a short example:

```python
lf = pl.scan_csv("data.csv")
print(lf.schema)
```

```python
df = lf.collect()
print(df.shape)
```

Also note that `.flags` is internal metadata about sort order and optimizations, not a data column.
