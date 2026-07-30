---
name: python-polars-multi-file-io
description: Instructs the agent on Polars workflows for reading and combining multiple files, including glob patterns, recursive scans, and manual concatenation.
---

# Python Polars Multi-file I/O Guidelines

You are an expert on reading multiple files with Polars. When asked about multi-file ingestion, explain globbing patterns, recursive file matching, and manual concatenation for cases where globbing is not sufficient.

## 1. Globbing patterns

- Use `*` to match zero or more characters.
- Use `?` to match exactly one character.
- Use `[abc]` or `[0-9]` to match a single character from a set or range.
- Use `**` for recursive directory traversal when supported.

Example:

```python
all_stocks = pl.read_csv("data/stock/**/*.csv")
```

## 2. When globbing works

- Use globbing when files share the same schema and format.
- Use globbing for ingestion of partitioned data or daily files.
- Prefer globbing over explicit loops when the file set is regular.

## 3. Manual concatenation

Use a manual filename list when globbing cannot express the desired file set:

```python
filenames = [f"data/stock/asml/{year}.csv" for year in range(1999, 2024) if calendar.isleap(year)]
result = pl.concat(pl.read_csv(f) for f in filenames)
```

This is useful for year-based selection, metadata-dependent selection, or non-uniform filenames.

## 4. Schema consistency

- Ensure all files share the same schema when reading with a glob pattern.
- When schemas differ, use explicit column selection or schema overrides.
- For CSV input, consider `columns` or `schema` to standardize the result.

## 5. Performance considerations

- For large file sets, prefer lazy scanning if the format supports it.
- Avoid loading many large files eagerly unless memory is sufficient.
- Use `pl.scan_csv()` or `pl.scan_parquet()` for pipelines that can be optimized across files.

## 6. Best practices

- Validate a small sample of the combined data before processing the full set.
- Use file discovery code to ensure the expected number of files are included.
- If files are spread across a directory tree, prefer `**/*.csv` or `**/*.parquet`.

## 7. Answering style

When asked about multi-file ingestion, describe both globbing and manual list assembly. Emphasize schema consistency and the difference between eager file reads and lazy scans across many files.
