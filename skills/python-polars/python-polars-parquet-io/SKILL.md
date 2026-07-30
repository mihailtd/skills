---
name: python-polars-parquet-io
description: Instructs the agent on Polars Parquet I/O workflows, including eager and lazy reading, schema preservation, compression, and write options.
---

# Python Polars Parquet I/O Guidelines

You are an expert on Polars Parquet workflows. When asked about Parquet, explain why it is the recommended format for analytic workloads, how to read and write it efficiently, and when to choose PyArrow vs the Rust-native reader.

## 1. Parquet read functions

- Eager read: `pl.read_parquet()`
- Lazy scan: `pl.scan_parquet()`
- Schema inspection: `pl.read_parquet_schema()`

Prefer `pl.scan_parquet()` for large datasets or pipelines where early pruning or projection is valuable.

## 2. Common read options

- `source`: file path, directory, or file-like object.
- `columns`: list of columns to select.
- `n_rows`: stop after reading a fixed number of rows (`use_pyarrow=False` required).
- `use_pyarrow`: use PyArrow reader if needed for compatibility.

## 3. Parquet strengths

- Columnar storage with excellent read performance.
- Stores schema and nested types.
- Supports efficient selective reads of columns.
- Compresses data to save disk space.

## 4. When to use Parquet

- Large analytic datasets that benefit from column pruning.
- Workflows that read data repeatedly.
- Data that contains nested or typed structures.
- When schema consistency is important.

## 5. Writing Parquet

- Use `df.write_parquet("output.parquet")`.
- Common arguments:
  - `file`: destination path.
  - `compression`: `zstd`, `lz4`, `snappy`, or `uncompressed`.
  - `compression_level`: controls compression strength.
  - `use_pyarrow`: choose the PyArrow writer for compatibility if needed.

## 6. Best practices

- Prefer Parquet over CSV for large datasets whenever compatibility allows.
- Use the Rust-native reader by default for speed, and only switch to PyArrow for edge-case compatibility.
- When reading directories of Parquet files, use the directory path directly or `pl.scan_parquet("folder/*.parquet")`.
- Validate the schema and selected columns especially when reading multiple files.

## 7. Answering style

When asked about Parquet, emphasize that it is the performance-oriented default for analytics and explain the difference between eager `read_parquet()` and lazy `scan_parquet()`. Mention compression selection and the usual tradeoffs.
