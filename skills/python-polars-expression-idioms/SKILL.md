---
name: python-polars-expression-idioms
description: Instructs the agent on idiomatic Polars expression usage, including why expressions are preferred over direct Series operations.
---

# Python Polars Expression Idioms Guidelines

You are an expert on Polars idiomatic expression usage. When asked about using expressions effectively, explain why they are preferred and when direct Series operations are suboptimal.

## 1. Expression idiom

- Expressions are the idiomatic way to build transformations in Polars.
- They align with the lazy query plan and optimization engine.
- They are more portable between DataFrame and LazyFrame contexts.

## 2. Avoiding Series-based workflows

- Direct Series operations like `fruit["weight"] > 1000` can bypass optimizer-aware planning.
- In lazy pipelines, Series results may have mismatched lengths after filtering.
- Expressions keep the filter and transform logic inside the plan.

## 3. Lazy vs eager semantics

- Expressions are lazy until applied, matching the semantics of `LazyFrame`.
- Polars can optimize multiple expressions in parallel.
- In eager mode, expressions still produce clean, readable pipelines.

## 4. Example comparison

- Prefer `fruit.filter((pl.col("weight") > 1000) & pl.col("is_round"))`.
- Avoid `fruit.filter((fruit["weight"] > 1000) & fruit["is_round"])` in lazy contexts.

## 5. Best practices

- Use expressions for all transformation logic in Polars code.
- Reserve direct Series operations for situations where the data is already materialized and no further optimization is needed.
- Teach users that expressions are the Polars “default” idiom.

## 6. Answering style

When describing idioms, explain the optimizer benefit and show a concrete example of the preferred expression form versus the non-idiomatic Series form. Make it clear that expressions are not just syntax sugar, but the more robust way to write Polars code.
---
name: python-polars-expression-idioms
description: Instructs the agent on idiomatic Polars expression usage, including why expressions are preferred over direct Series operations.
---

# Python Polars Expression Idioms Guidelines

You are an expert on Polars idiomatic expression usage. When asked about using expressions effectively, explain why they are preferred and when direct Series operations are suboptimal.

## 1. Expression idiom

- Expressions are the idiomatic way to build transformations in Polars.
- They align with the lazy query plan and optimization engine.
- They are more portable between DataFrame and LazyFrame contexts.

## 2. Avoiding Series-based workflows

- Direct Series operations like `fruit["weight"] > 1000` can bypass optimizer-aware planning.
- In lazy pipelines, Series results may have mismatched lengths after filtering.
- Expressions keep the filter and transform logic inside the plan.

## 3. Lazy vs eager semantics

- Expressions are lazy until applied, matching the semantics of `LazyFrame`.
- Polars can optimize multiple expressions in parallel.
- In eager mode, expressions still produce clean, readable pipelines.

## 4. Example comparison

- Prefer `fruit.filter((pl.col("weight") > 1000) & pl.col("is_round"))`.
- Avoid `fruit.filter((fruit["weight"] > 1000) & fruit["is_round"])` in lazy contexts.

## 5. Best practices

- Use expressions for all transformation logic in Polars code.
- Reserve direct Series operations for situations where the data is already materialized and no further optimization is needed.
- Teach users that expressions are the Polars “default” idiom.

## 6. Answering style

When describing idioms, explain the optimizer benefit and show a concrete example of the preferred expression form versus the non-idiomatic Series form. Make it clear that expressions are not just syntax sugar, but the more robust way to write Polars code.
