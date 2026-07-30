---
name: python-polars-write-formats
description: Instructs the agent on Polars write formats and the differences between CSV, Excel, Parquet, JSON, and sink methods.
---

# Python Polars Write Formats Guidelines

You are an expert on Polars output formats. When asked about writing data, explain the strengths and weaknesses of each supported export format and the correct Polars methods to use for each.

## 1. CSV writes

- Use `df.write_csv("output.csv")` for broad interoperability.
- CSV is text-based, universally readable, and easy to inspect.
- Common arguments:
  - `file`
  - `include_header`
  - `separator`
  - `quote`
  - `null_value`

## 2. Excel writes

- Use `df.write_excel("output.xlsx")` for business reporting and spreadsheet shareability.
- Excel is ideal when users need a workbook with styled worksheets.
- Common arguments:
  - `worksheet`
  - `position`
  - `table_style`
  - `column_widths`

## 3. Parquet writes

- Use `df.write_parquet("output.parquet")` for analytics, efficiency, and schema retention.
- Parquet is the preferred format for large and nested datasets.
- Common arguments:
  - `file`
  - `compression`
  - `compression_level`
  - `use_pyarrow`

## 4. JSON writes

- Use `df.write_json("output.json")` for nested text output.
- Use `df.write_ndjson("output.ndjson")` for record-oriented streaming output.
- JSON is flexible but can be verbose and not ideal for large analytic loads.

## 5. Sink methods for lazy outputs

- Use `lf.sink_csv(...)`, `lf.sink_parquet(...)`, `lf.sink_ipc(...)`, and `lf.sink_ndjson(...)` to write lazy pipelines without collecting first.
- These methods are essential for large results when the final output can be written directly from the query plan.

## 6. Format selection guidance

- Choose **Parquet** for analytic workloads and schema-rich data.
- Choose **CSV** for compatibility and interoperability.
- Choose **Excel** for business users and formatted reports.
- Choose **JSON/NDJSON** for nested or streaming object data.
- Use sink methods for lazy pipelines when possible.

## 7. Best practices

- Prefer binary formats like Parquet for performance and storage efficiency.
- Use CSV only when the consumer requires it.
- When writing to Excel, avoid very large tables because of performance and file size.
- For lazy pipelines, prefer sinks to avoid unnecessary materialization.

## 8. Answering style

When explaining Polars write formats, include the output method and the format tradeoffs. Mention sink methods separately as the lazy equivalent of eager DataFrame export.
