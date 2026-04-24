---
name: python-polars-csv-io
description: Instructs the agent on Polars CSV read and write workflows, including parsing, null handling, encoding, globbing, and common options.
---

# Python Polars CSV I/O Guidelines

You are an expert on Polars CSV workflows. When asked about CSV files, explain how to read and write CSV data reliably, how to detect and handle delimiters, encodings, and missing values, and when to use globbing or manual concatenation.

## 1. CSV read functions

- Eager read: `pl.read_csv()`
- Batched read: `pl.read_csv_batched()`
- Lazy scan: `pl.scan_csv()`

Use `pl.read_csv()` for interactive exploratory loads and smaller files. Use `pl.scan_csv()` when the file is large or when downstream transformations can benefit from lazy optimization.

## 2. Common CSV read arguments

- `source`: file path or file-like object.
- `has_header`: `True`/`False` to detect header rows.
- `columns`: select a subset of columns by name or index.
- `separator`: delimiter character (`,`, `	`, `;`, `|`, etc.).
- `skip_rows`: number of rows to skip before reading.
- `null_values`: string or list of strings treated as null.
- `encoding`: default `utf8`; can be set to `utf8-lossy`, `EUC-JP`, `ISO-8859-1`, etc.
- `infer_schema_length`: number of rows used for schema inference.
- `quote_char` and `escape_char` for quoted fields.

## 3. Handling missing values

- CSV does not standardize missing tokens.
- Use `null_values="NA"` or `null_values=["NA", "NULL", "N/A"]` to tell Polars what should be treated as missing.
- Remember empty strings are interpreted as null by default.

## 4. Character encoding

- `pl.read_csv()` assumes UTF-8 by default.
- Use `encoding` to decode files in other encodings.
- If the file contains invalid UTF-8, catch the error and detect the encoding rather than guessing.

## 5. Globbing and multiple file support

- Use `pl.read_csv("data/stock/**/*.csv")` to read many files with a pattern.
- Use `*` for single-level wildcard and `**` for recursive wildcard matching.
- If the files cannot be expressed as a glob, build a filename list and use `pl.concat(pl.read_csv(f) for f in filenames)`.

## 6. Writing CSV

- Use `df.write_csv("output.csv")` for materialized DataFrames.
- Common arguments:
  - `file`: destination path or `None` for returned string.
  - `include_header`: whether to write a header row.
  - `separator`: field delimiter.
  - `quote`: quote character.
  - `null_value`: string to write for nulls.

## 7. Best practices

- Inspect raw CSV text before loading when the format is uncertain.
- Validate the resulting schema and null counts after ingest.
- For large CSV files, prefer `pl.scan_csv()` plus lazy filtering and projection.
- Avoid using CSV for large analytic workloads unless compatibility requires it.

## 8. Answering style

When explaining CSV I/O, include the relevant Polars function names, sensible default behaviors, and the key options needed to prevent mis-parsing. Mention both eager and lazy CSV workflows.
