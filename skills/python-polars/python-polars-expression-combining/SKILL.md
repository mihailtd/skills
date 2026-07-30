---
name: python-polars-expression-combining
description: Instructs the agent on combining Polars expressions using arithmetic, comparisons, Boolean algebra, bitwise operations, and module-level functions.
---

# Python Polars Expression Combining Guidelines

You are an expert on combining Polars expressions. When asked about Chapter 9, explain how to build new expressions from multiple existing ones and when to use inline operators, Expr methods, or module-level functions.

## 1. Combining expressions

- Combining expressions is necessary when a result depends on more than one value, column, or intermediate expression.
- The most common cases are arithmetic between columns, Boolean filters, conditional logic, and horizontal aggregation across columns.
- The result is a new `pl.Expr` object that can be passed to `select`, `with_columns`, `filter`, `agg`, and other frame methods.

## 2. Inline operators versus methods

- Prefer inline operators when a corresponding operator exists: `+`, `-`, `*`, `/`, `//`, `**`, `%`, `&`, `|`, `^`, `~`, `<`, `<=`, `==`, `!=`, `>=`, `>`.
- Every inline operator has a method equivalent, e.g. `Expr.add(...)`, `Expr.mul(...)`, `Expr.gt(...)`, `Expr.and_(...)`.
- Use method forms when the inline operator is not available, when you want a more explicit style, or when you need method-specific arguments.
- Wrap combined expressions in parentheses before continuing the chain, e.g. `(pl.col("i") * pl.col("j")).alias("product")`.

## 3. Arithmetic operations

- Common arithmetic combinations include addition, subtraction, multiplication, division, floor division, exponentiation, and modulus.
- Use inline operators for readability: `pl.col("a") + pl.col("b")`, `pl.col("x") / 1000`, `pl.col("i") ** pl.col("j")`.
- Use the method equivalents when needed: `.add()`, `.sub()`, `.mul()`, `.truediv()`, `.floordiv()`, `.pow()`, `.mod()`.
- For vector operations like dot product, use `Expr.dot(...)` because there is no inline operator.

## 4. Comparison operations

- Comparison operators produce Boolean expressions: `<`, `<=`, `==`, `>=`, `>`, `!=`.
- Comparison methods are `Expr.lt(...)`, `Expr.le(...)`, `Expr.eq(...)`, `Expr.ge(...)`, `Expr.gt(...)`, `Expr.ne(...)`.
- Do not chain comparisons like Python: `pl.col("x") < pl.lit(3) < pl.lit(5)` is invalid.
- Combine multiple comparisons with Boolean operators: `(pl.col("x") > 3) & (pl.col("x") < 5)`.
- Use `Expr.is_between(...)` when available for range checks instead of manual chained comparisons.

## 5. Boolean algebra

- Use `&` for logical AND, `|` for logical OR, and `~` for logical NOT.
- The equivalent methods are `.and_(...)`, `.or_(...)`, and `.not_()`.
- Exclusive OR is `^` or `Expr.xor(...)`.
- Boolean expressions are a common way to combine filters, construct conditional logic, and build truth tables.
- For complex conditions, prefer clear grouping with parentheses and descriptive aliases.

## 6. Bitwise operations

- The same operators `&`, `|`, `^`, and `~` also work on integer expressions as bitwise operators.
- Use Booleans when an expression is meant to represent truth values, and use integer types when the intent is bitwise manipulation.
- The methods are the same as the Boolean methods, but the semantics are bitwise for integer inputs.

## 7. Module-level combining functions

- Polars provides functions that combine multiple expressions without relying on inline operators.
- Common functions include:
  - `pl.concat_list(...)`
  - `pl.concat_str(...)`
  - `pl.format(...)`
  - `pl.struct(...)`
  - `pl.coalesce(...)`
  - `pl.all_horizontal(...)`
  - `pl.any_horizontal(...)`
  - `pl.sum_horizontal(...)`, `pl.max_horizontal(...)`, `pl.min_horizontal(...)`
  - `pl.when(...).then(...).otherwise(...)`
- Use these when you need horizontal combination of many columns, string formatting, struct/list construction, or conditional logic.

## 8. Practical guidance

- Emphasize readability by using inline operators when they exist and methods when they are more explicit.
- Avoid ambiguous Python semantics by keeping Polars expressions separate from Python logical operators like `and`, `or`, and `not`.
- Use `pl.col(...)` with names, regex selectors, or typed selectors to reference columns consistently.
- Explain that combining expressions is a continuation of the expression-based style from previous chapters, not a separate workflow.
- If the user asks for details on a specific operator or function, cite the relevant Polars method and include a short example.

## 9. Answering style

When answering Chapter 9 questions, do the following:

- Frame the question as building a new `pl.Expr` from one or more inputs.
- Clarify whether the combination is arithmetic, comparison, Boolean, bitwise, or function-based.
- Provide one short example that uses the appropriate inline operator or function.
- Mention any pitfalls such as comparison chaining, mixing incompatible dtypes, or losing expression laziness.
- Point users to the Polars API reference for full method coverage while keeping the answer focused on how expressions are combined.
