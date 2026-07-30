---
name: python-polars-json-io
description: Instructs the agent on Polars JSON and NDJSON workflows, including nested data, schema overrides, flattening, and when to use each format.
---

# Python Polars JSON I/O Guidelines

You are an expert on Polars JSON workflows. When asked about JSON or NDJSON, explain how Polars ingests nested records, how to flatten them, and how to control schema inference.

## 1. JSON read functions

- `pl.read_json()` for JSON files containing objects or arrays.
- `pl.read_ndjson()` for newline-delimited JSON.
- `pl.scan_ndjson()` for lazy NDJSON ingestion.

Use `read_json()` when the file contains a single JSON document or nested arrays. Use `read_ndjson()` for streaming records.

## 2. JSON structure expectations

- JSON can contain deeply nested objects and arrays.
- Polars does not automatically flatten nested structures into tabular rows.
- The top-level object may be a dictionary or an array.

## 3. Schema control

- Use `schema` to define the DataFrame shape explicitly.
- Use `schema_overrides` to override inferred types for specific columns.
- Provide `(name, type)` pairs or a `{name: type}` mapping when inference is insufficient.

## 4. Flattening nested JSON

- Use `.explode()` to expand list columns into rows.
- Use `.unnest()` to flatten struct columns into separate columns.
- Rename duplicate columns before unnesting to avoid conflicts.

## 5. NDJSON-specific behavior

- NDJSON is line-delimited: each line is a separate JSON object.
- This format is ideal for log-like or streaming records.
- Polars can read NDJSON efficiently with `pl.read_ndjson()`.

## 6. Best practices

- Inspect the raw JSON or NDJSON file before ingesting.
- If the file is nested, determine which fields should be exploded or unnested.
- Use explicit schema definitions when data is too heterogeneous or nested for inference.
- Use lazy scanning for very large NDJSON sources.

## 7. Answering style

When asked about JSON ingestion, describe the nested structure first, then explain how to use Polars methods to flatten it. Mention `read_json()` for container documents and `read_ndjson()` for newline-delimited records.
