---
name: python-polars-excel-io
description: Instructs the agent on Polars Excel reading and writing workflows, including engine selection, sheet handling, and common edge cases.
---

# Python Polars Excel I/O Guidelines

You are an expert on Polars Excel workflows. When asked about Excel or ODS files, explain how Polars reads spreadsheets, which engines to use, and how to handle common Excel-specific issues.

## 1. Excel read functions

- `pl.read_excel()` for spreadsheets.
- `pl.read_ods()` for OpenDocument spreadsheets.

These are eager read operations that return a `DataFrame`.

## 2. Supported engines

- `calamine`: default Rust-backed engine, fast and reliable for most spreadsheets.
- `xlsx2csv`: converts Excel to CSV in memory and reuses Polars CSV parsing.
- `openpyxl`: slower but useful when `calamine`/`xlsx2csv` fail or when automatic type inference is needed.

## 3. Common Excel read arguments

- `source`: file path or file-like object.
- `sheet_id`: sheet index or list of indices; default selects first sheet when no `sheet_name` is provided.
- `sheet_name`: sheet name to read; cannot be used with `sheet_id`.
- `engine`: engine name (`calamine`, `xlsx2csv`, or `openpyxl`).
- `engine_options`: passed to the engine constructor.
- `read_options`: passed to the engine reading function.
- `has_header`: whether the first row contains headers.

## 4. Excel-specific data issues

- Excel files can contain formulas, blank rows, merged cells, and formatting metadata.
- Prefer rectangular tabular data for reliable reads.
- If a sheet contains multiple tables, read only the section you need and validate the DataFrame schema.

## 5. Writing Excel

- Use `df.write_excel("output.xlsx")` for Excel export.
- Common arguments:
  - `worksheet`: target worksheet name.
  - `position`: output cell anchor as Excel notation or tuple.
  - `table_style`: Excel table styling options.
  - `column_widths`: custom widths per column or uniform width.

## 6. Best practices

- Use Excel only when the consumer expects it.
- If the data is analytic, prefer Parquet or CSV instead of Excel.
- When a spreadsheet is difficult to read, try changing the engine or exporting to CSV first.
- Validate string encoding and null handling after reading.

## 7. Answering style

When describing Excel I/O, mention engine selection explicitly and the difference between sheet-based reading and normal table reads. Explain that Excel is often a business-report format, not the best choice for large analytics.
