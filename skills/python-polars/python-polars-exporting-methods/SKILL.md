---
name: python-polars-exporting-methods
description: Instructs the agent on Polars exporting methods and the difference between eager DataFrame exports and LazyFrame terminal sinks.
---

# Python Polars Exporting Methods Guidelines

You are an expert on Polars exporting methods. When asked about exporting results, explain which operations are available on `DataFrame`, what they do, and how lazy pipelines reach the same goal.

## 1. DataFrame export methods

The `DataFrame` API supports exporting materialized data to many formats:

- `.to_arrow(...)`
- `.to_dict(...)`
- `.to_dicts()`
- `.to_init_repr(...)`
- `.to_jax(...)`
- `.to_numpy(...)`
- `.to_pandas(...)`
- `.to_struct(...)`
- `.to_torch(...)`
- `.serialize(...)`

These methods convert the `DataFrame` into another in-memory representation or serialized payload.

## 2. LazyFrame terminal sinks

Since `LazyFrame` has no data until collection, exporting is done by collecting or sinking:

- `.collect()` / `.collect_async()`
- `.collect_schema()` to retrieve schema only
- `.sink_csv(...)`
- `.sink_ipc(...)`
- `.sink_parquet(...)`
- `.sink_ndjson(...)`

Use these methods when you want to write the result from a lazy pipeline directly to disk or cloud storage.

## 3. Use cases

- Use `df.to_pandas()` or `df.to_numpy()` when integrating with Python ML or visualization libraries.
- Use `df.serialize()` for portable binary storage.
- Use `lf.sink_parquet()` / `lf.sink_csv()` to avoid loading the full result into memory.
- Use `lf.collect()` when you need a concrete DataFrame after a lazy pipeline.

## 4. Practical guidance

- Emphasize that DataFrame exports are for materialized results.
- Emphasize that LazyFrame sinks and `collect()` are terminal operations that trigger execution.
- Recommend `.collect_schema()` when you need output schema validation without full computation.

## 5. Answering style

When describing Polars export methods, mention the difference between exporting data from a concrete frame and terminalizing a lazy plan. Provide examples for the right export method depending on the workflow.
