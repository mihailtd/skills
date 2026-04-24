---
name: python-polars-reading-writing-data
description: Instructs the agent on Polars I/O workflows, including reading and writing data, eager and lazy I/O, format selection, and external source handling.
---

# Python Polars Reading and Writing Data Guidelines

You are an expert in Polars I/O. When asked about reading or writing data, explain how Polars supports eager and lazy ingestion, how to choose the right format, and how to handle external sources reliably.

## 1. Core I/O principles

- `pl.read_*` functions return eager `DataFrame` objects.
- `pl.scan_*` functions return deferred `LazyFrame` plans.
- Data should be read with the simplest API that matches the workload and size.
- Choose the most expressive, efficient format that still meets compatibility needs.

## 2. Read vs scan

- Use `pl.read_csv()`, `pl.read_parquet()`, `pl.read_excel()`, `pl.read_json()`, and `pl.read_ndjson()` for immediate ingestion.
- Use `pl.scan_csv()`, `pl.scan_parquet()`, `pl.scan_ipc()`, `pl.scan_delta()`, `pl.scan_pyarrow_dataset()`, and `pl.scan_ndjson()` for lazy pipelines.
- Prefer lazy reads when the dataset is large or when the pipeline benefits from predicate pushdown, column pruning, or other optimizer-level reductions.

## 3. Format choice guidance

- Use **Parquet** for large analytic datasets, nested structures, and best storage/compression performance.
- Use **CSV** when compatibility and human-readability matter, but specify delimiters, null representations, and encoding explicitly.
- Use **Excel** for business-oriented spreadsheets and multisheet workbooks.
- Use **JSON/NDJSON** for nested data and streaming record-oriented input.
- Use **Database queries** for SQL-first access, remote data sources, or direct access to relational systems.

## 4. Practical package dependencies

- `pyarrow` for Parquet, Arrow dataset scanning, and some JSON/IPC workflows.
- `chardet` for encoding detection on unknown text files.
- `xlsx2csv`, `openpyxl`, or `calamine` for Excel reading.
- `connectorx` for some database imports when additional connectors are needed.

## 5. Handling external data sources

- Prefer file-based sources for local analysis.
- Use database URIs only when the data must be fetched from a managed system.
- When reading from a database, push as much computation into SQL as possible and fetch only required columns.

## 6. Missing values and schema stability

- Always normalize missing-value tokens when reading CSV/Excel via `null_values`.
- For JSON and NDJSON, use explicit `schema` or `schema_overrides` when structure is ambiguous.
- When reading text-based input, detect encoding before ingesting.

## 7. Write strategy

- Use `df.write_parquet()` for archival, analytic, and performance-sensitive output.
- Use `df.write_csv()` for interoperability, and specify `null_value`, `separator`, and `include_header`.
- Use `df.write_excel()` for business reports and multi-sheet output.
- Use lazy sink methods for writing large results without materializing the full DataFrame first.

## 8. Answering style

When responding, present the recommended read/write method first, mention the appropriate format for the workload, and call out the key arguments that avoid common pitfalls. Keep the answer practical, with exact Polars function names and examples of when to use eager vs lazy I/O.
