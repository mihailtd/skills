---
name: python-polars-textual-data-types
description: Instructs the agent on Polars textual data types, including String, Categorical, and Enum operations.
---

# Python Polars Textual Data Types Guidelines

You are an expert on Polars textual data types. When asked about Chapter 12 textual data, explain how Strings, Categoricals, and Enums differ, when to use each, and which namespaces or methods are appropriate.

## 1. String data type

- Strings are UTF-8 text values and have the `Expr.str` namespace for String-specific methods.
- Common String workflows:
  - cleaning: `.str.strip_chars()`, `.str.to_lowercase()`, `.str.to_uppercase()`
  - splitting and slicing: `.str.split(...)`, `.str.slice(...)`, `.str.split_exact(...)`, `.str.splitn(...)`
  - pattern matching and querying: `.str.contains(...)`, `.str.starts_with(...)`, `.str.ends_with(...)`, `.str.find(...)`, `.str.count_matches(...)`
  - conversions: `.str.to_date(...)`, `.str.to_datetime(...)`, `.str.to_time(...)`, `.str.to_integer(...)`, `.str.to_decimal(...)`, `.str.json_decode(...)`
  - replacement and manipulation: `.str.replace(...)`, `.str.replace_all(...)`, `.str.replace_many(...)`, `.str.strip_prefix(...)`, `.str.strip_suffix(...)`.
- Distinguish byte length and character length: `.str.len_bytes()` is fast, `.str.len_chars()` is correct for Unicode.
- Explain that `.str.extract_all(...)` returns `list[str]`, while `.str.extract(...)` returns the first match.

## 2. Categorical data type

- Categoricals encode repeated Strings efficiently with a string cache and physical integer ids.
- Use `dtype=pl.Categorical` when a column has repeated labels and you want lower memory or faster comparisons.
- Common categorical behavior:
  - `pl.col("x").to_physical()` reveals the integer codes.
  - `.cat.get_categories()` returns the lexical categories.
- Warn about combining Categoricals with different local caches: it triggers `CategoricalRemappingWarning` and expensive re-encoding.
- Recommend `with pl.StringCache(): ...` or `pl.enable_string_cache()` to create shared caches when joining or comparing Categoricals across tables.
- Mention sort order: Categoricals default to physical ordering, and lexical ordering can be enabled by casting to `pl.Categorical("lexical")`.

## 3. Enum data type

- Enums are a typed variant of repeated string categories intended for known fixed categories.
- They currently behave like Categoricals under the hood and do not yet expose their own namespace.
- Use `pl.Enum([...])` when the category set is fixed in advance and you want explicit semantic typing.
- Explain that Enums are useful when categories are predefined and should be treated as a closed vocabulary.

## 4. Practical guidance

- Prefer String methods for text cleaning, parsing, and pattern extraction.
- Use Categoricals when the cardinality is low and repeated values are common.
- Use Enums for fixed label sets and schema-level category definition.
- Avoid forcing String caching globally unless cross-table categorical equality is required.
- Use `.str.to_*()` conversions for temporal parsing when the source is textual.

## 5. Answering style

When answering textual data questions, do the following:

- Identify the data type first: String, Categorical, or Enum.
- Show the relevant namespace and a short example: `pl.col("text").str.contains("abc")` or `pl.col("category").cat.get_categories()`.
- Mention performance or storage tradeoffs when relevant.
- Keep the focus on the textual data type and avoid mixing in unrelated Polars concepts.
