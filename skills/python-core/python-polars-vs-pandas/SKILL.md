---
name: python-polars-vs-pandas
description: Instructs the agent to use Polars instead of pandas for Python data processing — a Rust-based, multi-threaded execution engine versus pandas' single-threaded eager execution, Apache Arrow memory layout versus NumPy-array copies, a lazy query optimizer that can push down filters/projections before running, streaming support for larger-than-RAM datasets, and a readable expression-based API. Use when writing new data-processing code, reviewing pandas code, or explaining why this project doesn't use pandas.
---

# Data Processing — Polars over pandas

This is a team standard: **use Polars, not pandas, for Python data
processing in this project.** This skill explains why, gives the concrete
mechanical differences behind the claim, and shows the syntax difference on
a representative pipeline. For Polars' own mechanics once you've adopted
it (expressions, I/O, lazy API details, aggregation), see the `python-polars`
package — this skill covers the *decision*, not the full API.

---

## When to use this skill

Use this skill when you need to:

- write new data-processing, ETL, or analysis code and need to justify or
  apply the default library choice,
- review code that imports or uses `pandas`, and decide whether to flag it,
- explain to a teammate, new hire, or stakeholder why this project doesn't
  use pandas,
- port existing pandas code to Polars,
- decide how to handle a third-party dependency that strictly requires a
  pandas `DataFrame`.

---

## Outcome

Produce data-processing work that:

- uses `polars`, not `pandas`, for any new DataFrame-based code in this
  project,
- prefers the lazy API (`pl.scan_*` / `.lazy()` + `.collect()`) for
  pipelines with multiple chained operations, so the query optimizer can
  do its job, rather than defaulting to eager execution out of habit,
- is written as expression-based chains rather than imperative,
  index-heavy manipulation,
- only touches pandas at an explicit interoperability boundary (a
  third-party library that requires it), converting immediately rather
  than letting pandas usage spread through the codebase,
- can be justified concretely — in terms of execution model, memory
  layout, and scale — not just asserted as a preference.

---

## Instructions for the AI

1. **Default to Polars for all new data-processing code**
   - When asked to load, transform, filter, aggregate, or export tabular
     data in Python, write it with `polars`, not `pandas`.
   - If existing code uses `pandas`, treat it as a candidate for
     conversion rather than something to extend in place — flag `import
     pandas` during review as a deviation from the team standard.
   - Install with `uv add polars` (see python-project-setup), consistent
     with the project's standard package manager.

2. **Explain the justification in concrete, mechanical terms**
   - **Execution engine:** Polars is written from scratch in Rust and
     multi-threads its workflows by default, using all available CPU
     cores in parallel. pandas executes eagerly, step-by-step, on a
     single core (its C extensions accelerate individual operations, but
     don't parallelize the pipeline as a whole). This is the direct
     source of Polars' raw speed advantage on realistic multi-step
     pipelines.
   - **Memory layout:** Polars uses the Apache Arrow columnar memory
     specification, which allows data to be shared between operations
     (and even between processes/languages) without expensive copying.
     pandas is built on NumPy arrays, and many common operations
     (filtering, joining, reshaping) create intermediate copies of the
     data, which costs both memory and time as datasets grow.
   - **Evaluation strategy:** Polars supports both eager execution (like
     pandas, useful for exploration) and lazy execution via `.lazy()` —
     lazy mode builds a query plan across the *entire* chain of
     operations before running anything, then optimizes it: reordering
     filters to run earlier, dropping columns that are never used
     (projection pushdown), and combining steps where possible. pandas
     has no equivalent — every operation runs immediately, in the exact
     order written, with no opportunity for the library itself to
     optimize the overall pipeline.
   - **Dataset scale:** because of the above, Polars comfortably handles
     multi-gigabyte datasets, including via streaming execution that
     processes data in chunks from disk without requiring the full
     dataset to fit in RAM. pandas is realistically limited to datasets
     that fit comfortably in memory — pushing past that means manual
     chunking workarounds pandas doesn't support natively.
   - **Syntax and readability:** Polars uses an expression-based API
     (`pl.col(...)` chains) that reads as a declarative pipeline and
     composes cleanly — multiple aggregations can be expressed and run in
     parallel within a single `.agg(...)` call. pandas frequently requires
     imperative, index-based manipulation (`.loc`, `.iloc`, chained
     boolean masks) that's harder to read and easier to get subtly wrong.
   - When asked to justify the standard, lead with the execution engine
     and lazy-optimization points — they're the direct cause of the
     performance difference — and use the memory-layout and dataset-scale
     points to explain *why* Polars remains viable as data grows, which
     pandas structurally cannot promise.

3. **Prefer the lazy API for multi-step pipelines**
   - For any pipeline with more than a couple of chained operations
     (filter, group-by, join, multiple derived columns), default to
     `pl.scan_csv` / `pl.scan_parquet` (or `df.lazy()` on an existing
     eager frame) and end the chain with `.collect()` — this is what
     actually lets Polars' query optimizer do its job (predicate
     pushdown, projection pushdown, reordering).
   - Reserve the eager API for quick, one-off exploration or small,
     single-operation scripts where the optimization overhead isn't worth
     invoking — see python-polars-eager-api and python-polars-lazy-api for
     the detailed guidance on choosing between them.

4. **Write expression-based, not imperative, transformations**
   - Chain operations using Polars expressions (`pl.col("x").filter(...)`,
     `.group_by(...).agg(...)`) rather than reaching for index-based or
     row-by-row manipulation.
   - Show the syntax difference when it clarifies the justification for a
     teammate coming from pandas:
     ```python
     # pandas — eager, imperative, step-by-step
     df_filtered = df[df["age"] > 30]
     result_pandas = df_filtered.groupby("department").agg({"salary": "mean"})

     # Polars — lazy, expression-based, optimizer-ready
     result_polars = (
         df.lazy()
         .filter(pl.col("age") > 30)
         .group_by("department")
         .agg(pl.col("salary").mean())
         .collect()
     )
     ```
   - Note for teammates translating pandas habits: `.loc`/`.iloc`-style
     indexing has no direct Polars equivalent by design — the fix is to
     re-express the operation as a filter/select expression, not to hunt
     for a one-to-one method replacement.

5. **Handle pandas interoperability at an explicit boundary**
   - When a third-party library strictly requires a pandas `DataFrame`
     (e.g., a specific plotting or ML library API), convert at the last
     possible moment with `polars_df.to_pandas()`, keeping the rest of the
     pipeline in Polars.
   - Don't let a single forced interop point become a reason to do the
     surrounding pipeline in pandas as well — isolate the conversion, not
     the library choice.

---

## Decision points and guidance

- **Is this new data-processing code, or existing pandas code?** New code
  is always Polars; existing pandas code is a conversion candidate, not
  something to extend.
- **Does the pipeline chain multiple operations?** If so, use the lazy API
  so the optimizer can actually help — eager Polars on a multi-step chain
  gives up most of the performance advantage.
- **Is the dataset larger than comfortably fits in RAM, or likely to grow
  that way?** This is one of the clearest, most concrete justifications for
  Polars over pandas — cite it directly.
- **Is pandas required by a third-party dependency?** Convert at that exact
  boundary with `.to_pandas()`; don't let it justify using pandas more
  broadly.

---

## Quality criteria

A strong Polars-over-pandas response should ensure that:

- **new code uses Polars**, with no new `import pandas` introduced without
  a specific, named interoperability reason,
- **multi-step pipelines use the lazy API** so query optimization actually
  applies, not just the library name,
- **transformations are expression-based**, not translated pandas idioms
  (`.loc`/`.iloc` patterns re-implemented awkwardly in Polars),
- **the justification, when asked for, is mechanical and specific**
  (execution engine, memory layout, lazy optimization, streaming) rather
  than a bare assertion that "Polars is faster,"
- **pandas interop, where unavoidable, is isolated** to a single explicit
  conversion point.

---

## Example prompts

- "Write a data pipeline that filters and aggregates this dataset."
- "This module uses pandas — help me convert it to Polars."
- "Why does our team use Polars instead of pandas? I need to explain this
  to a new teammate."
- "This third-party plotting library needs a pandas DataFrame — how should
  I handle that without pulling pandas into the rest of the pipeline?"

---

## Related guidance

Use this tool alongside:

- python-polars-eager-api
- python-polars-lazy-api
- python-polars-expressions-overview
- python
