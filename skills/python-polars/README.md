# Python — Polars

Polars-specific skills covering expressions, I/O (CSV/Parquet/JSON/Excel/database), aggregation, groupby, reshaping, and the lazy API.

For data-processing/ETL projects built on Polars. Pair with `python-core`.

## Install

```bash
npx skills add mihailtd/skills/skills/python-polars --all
```

Add `-g` to install globally instead of per-project. Use `--skill <name>` (repeatable) instead of `--all` to cherry-pick individual skills from this package, e.g.:

```bash
npx skills add mihailtd/skills/skills/python-polars --skill python-polars-aggregation-methods
```

## Skills (36)

- **[python-polars-aggregation-methods](python-polars-aggregation-methods/SKILL.md)** — Instructs the agent on Polars aggregation methods for DataFrame and LazyFrame, including vertical and horizontal aggregation semantics.
- **[python-polars-attributes-methods](python-polars-attributes-methods/SKILL.md)** — Instructs the agent on Polars DataFrame and LazyFrame attribute methods and their availability across APIs.
- **[python-polars-column-operations](python-polars-column-operations/SKILL.md)** — Instructs the agent on selecting and creating columns in Polars, including selectors, df.select(), df.with_columns(), and related column operations.
- **[python-polars-computation-methods](python-polars-computation-methods/SKILL.md)** — Instructs the agent on Polars DataFrame computation methods and their availability relative to LazyFrame.
- **[python-polars-csv-io](python-polars-csv-io/SKILL.md)** — Instructs the agent on Polars CSV read and write workflows, including parsing, null handling, encoding, globbing, and common options.
- **[python-polars-data-structures](python-polars-data-structures/SKILL.md)** — Instructs the agent on Polars data structures, including Series, DataFrame, LazyFrame, and nested column formats, with guidance for practical usage and performance-aware design.
- **[python-polars-data-types](python-polars-data-types/SKILL.md)** — Instructs the agent on Polars data types, how they map to Arrow semantics, and how to work with casting, missing values, and performance-aware type selection.
- **[python-polars-database-io](python-polars-database-io/SKILL.md)** — Instructs the agent on Polars database connectivity and query workflows, including URI formats, query pushdown, and using databases as external sources.
- **[python-polars-descriptive-methods](python-polars-descriptive-methods/SKILL.md)** — Instructs the agent on Polars descriptive methods for DataFrame and LazyFrame, including plan visualization and summary statistics.
- **[python-polars-eager-api](python-polars-eager-api/SKILL.md)** — Instructs the agent on Polars eager API usage with DataFrame, immediate execution semantics, and best practices for exploration and iterative analysis.
- **[python-polars-encoding-io](python-polars-encoding-io/SKILL.md)** — Instructs the agent on Polars character encoding handling for reading text data, including detection, common encodings, and UTF-8 fallbacks.
- **[python-polars-excel-io](python-polars-excel-io/SKILL.md)** — Instructs the agent on Polars Excel reading and writing workflows, including engine selection, sheet handling, and common edge cases.
- **[python-polars-exporting-methods](python-polars-exporting-methods/SKILL.md)** — Instructs the agent on Polars exporting methods and the difference between eager DataFrame exports and LazyFrame terminal sinks.
- **[python-polars-expression-combining](python-polars-expression-combining/SKILL.md)** — Instructs the agent on combining Polars expressions using arithmetic, comparisons, Boolean algebra, bitwise operations, and module-level functions.
- **[python-polars-expression-creation](python-polars-expression-creation/SKILL.md)** — Instructs the agent on creating Polars expressions from columns, literals, and ranges, including `pl.col`, `pl.lit`, and range constructors.
- **[python-polars-expression-filtering](python-polars-expression-filtering/SKILL.md)** — Instructs the agent on filtering rows with Polars expressions, including comparison operators, boolean logic, and expression composition.
- **[python-polars-expression-groupby-aggregation](python-polars-expression-groupby-aggregation/SKILL.md)** — Instructs the agent on Polars GroupBy and aggregation expressions, including group keys, summaries, and expression reuse.
- **[python-polars-expression-idioms](python-polars-expression-idioms/SKILL.md)** — Instructs the agent on idiomatic Polars expression usage, including why expressions are preferred over direct Series operations.
- **[python-polars-expression-naming](python-polars-expression-naming/SKILL.md)** — Instructs the agent on naming and renaming Polars expressions, including aliasing, keyword column naming, and Expr.name utilities.
- **[python-polars-expression-operations](python-polars-expression-operations/SKILL.md)** — Instructs the agent on continuing a Polars expression with additional methods, including element-wise transforms, missing-value handling, smoothing, selection, and summaries.
- **[python-polars-expression-ranges](python-polars-expression-ranges/SKILL.md)** — Instructs the agent on creating Polars range expressions for integers, dates, times, and datetimes.
- **[python-polars-expression-selection](python-polars-expression-selection/SKILL.md)** — Instructs the agent on Polars expression-based selection, including column selection by name, regex, type, and wildcard selectors.
- **[python-polars-expression-sorting](python-polars-expression-sorting/SKILL.md)** — Instructs the agent on sorting with Polars expressions, including expression-based sort keys and sort-related expression semantics.
- **[python-polars-expressions-overview](python-polars-expressions-overview/SKILL.md)** — Instructs the agent on Polars expression fundamentals, their lazy semantics, and how expressions are applied across DataFrames and LazyFrames.
- **[python-polars-groupby-methods](python-polars-groupby-methods/SKILL.md)** — Instructs the agent on Polars GroupBy methods for DataFrame and LazyFrame, including aggregation and iteration semantics.
- **[python-polars-json-io](python-polars-json-io/SKILL.md)** — Instructs the agent on Polars JSON and NDJSON workflows, including nested data, schema overrides, flattening, and when to use each format.
- **[python-polars-lazy-api](python-polars-lazy-api/SKILL.md)** — Instructs the agent on Polars lazy API usage with LazyFrame, deferred execution, query planning, and optimization strategies.
- **[python-polars-manipulation-methods](python-polars-manipulation-methods/SKILL.md)** — Instructs the agent on Polars DataFrame and LazyFrame manipulation methods, including selection, filtering, joins, reshaping, and transformations.
- **[python-polars-miscellaneous-methods](python-polars-miscellaneous-methods/SKILL.md)** — Instructs the agent on Polars miscellaneous methods for both DataFrame and LazyFrame, including collection, caching, profiling, and function mapping.
- **[python-polars-multi-file-io](python-polars-multi-file-io/SKILL.md)** — Instructs the agent on Polars workflows for reading and combining multiple files, including glob patterns, recursive scans, and manual concatenation.
- **[python-polars-parquet-io](python-polars-parquet-io/SKILL.md)** — Instructs the agent on Polars Parquet I/O workflows, including eager and lazy reading, schema preservation, compression, and write options.
- **[python-polars-reading-writing-data](python-polars-reading-writing-data/SKILL.md)** — Instructs the agent on Polars I/O workflows, including reading and writing data, eager and lazy I/O, format selection, and external source handling.
- **[python-polars-reshaping](python-polars-reshaping/SKILL.md)** — Instructs the agent on Polars reshaping methods, including pivot, unpivot, transpose, explode, and partition_by.
- **[python-polars-row-operations](python-polars-row-operations/SKILL.md)** — Instructs the agent on filtering and sorting rows in Polars, including row selection, sort semantics, slicing, sampling, and related operations.
- **[python-polars-textual-data-types](python-polars-textual-data-types/SKILL.md)** — Instructs the agent on Polars textual data types, including String, Categorical, and Enum operations.
- **[python-polars-write-formats](python-polars-write-formats/SKILL.md)** — Instructs the agent on Polars write formats and the differences between CSV, Excel, Parquet, JSON, and sink methods.
