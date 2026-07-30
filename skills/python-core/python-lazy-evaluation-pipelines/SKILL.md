---
name: python-lazy-evaluation-pipelines
description: >
  Instructs the agent to avoid materializing large datasets in memory by prioritizing
  lazy evaluation. Mandates the use of yield, yield from, generator expressions, stacked
  generator pipelines, map(), filter(), reduce(), and itertools for functional data
  processing. Use when processing collections, building data pipelines, or optimizing
  memory usage.
---

# Python Lazy Evaluation & Functional Pipelines Guidelines

You are an expert Python developer specializing in functional programming and efficient data transformation. When asked to process collections of data, apply transformations, or summarize datasets, you must prioritize lazy evaluation and avoid materializing large datasets in memory. Adhere to the following rules:

---

## 1. Generator Expressions over List Comprehensions

Avoid creating large, materialized in-memory data structures unless absolutely necessary.

*   Always prefer generator expressions `(x for x in iterable)` over list comprehensions `[x for x in iterable]`.
*   Generator expressions evaluate lazily, computing values only as requested, which drastically reduces memory footprint and improves initialization speed.
*   Only materialize a sequence (using `list()` or `tuple()`) if the application explicitly requires evaluating the size with `len()` or making multiple passes over the same data.

## 2. Generator Functions with `yield`

When writing functions that produce a sequence of values, do not accumulate the results in a local list state just to return them at the end.

*   Write generator functions using the `yield` statement.
*   This transforms the function into an iterator, returning values lazily without polluting the local namespace with large intermediate variables.

## 3. Delegating with `yield from`

When writing recursive generator functions, or when a generator needs to yield all values from another iterable, do not use a standard `for` loop to yield items individually.

*   Use the `yield from` expression to directly consume and yield values from a sub-generator or recursive call.
*   Never use `return recursive_iter(args)` inside a recursive generator, as it will break the iteration and return a generator object instead of yielding its values.

## 4. Map, Filter, and Reduce Patterns

Use standard functional patterns to transform data.

*   **Map:** To apply a transformation to every item, use the built-in `map(function, source)` or a generator expression `(function(x) for x in source)`.
*   **Filter:** To select a subset of items, use the built-in `filter(predicate, source)` or a generator expression `(x for x in source if predicate(x))`.
*   **Reduce:** To summarize a collection into a single value, use `functools.reduce(operator, source, initial)`. Remember that common reductions are already built-in, such as `sum()`, `any()`, `all()`, `max()`, `min()`, and `math.prod()`.

## 5. Stacked Generator Expressions (Pipelines)

Do not write monolithic loops that attempt to parse, filter, and summarize data all at once.

*   Decompose complex processing into small, independent generator functions or expressions.
*   Stack these generators to create a processing pipeline, where the result of one lazy generator serves directly as the input (source) to the next.

## 6. Leverage `itertools` for Complex Iteration

When you need complex map-reduce behavior or sophisticated iteration, use the `itertools` module.

*   Use functions like `itertools.takewhile()` to process items only until a certain condition is met, implementing early exit without stateful loops.
*   Avoid explicit stateful flags or breaking out of `for` loops manually when `itertools` provides a declarative alternative.

---

## Code Examples

**1. Generator Expressions vs. Materialization**

```python
from collections.abc import Iterator, Iterable

# BAD: Eagerly evaluates and creates a massive list in memory
def get_squared_eager(data: list[int]) -> list[int]:
    return [x**2 for x in data]

# GOOD: Lazily evaluates, returning an iterator that saves memory
def get_squared_lazy(data: Iterable[int]) -> Iterator[int]:
    return (x**2 for x in data)
```

**2. Generator Functions and `yield from`**

```python
from collections.abc import Iterator

# GOOD: Using yield to produce values lazily without building a list
def count_up_to(limit: int) -> Iterator[int]:
    for i in range(limit):
        yield i

# GOOD: Using yield from for recursion or delegating to another iterable
def bunch_of_numbers() -> Iterator[int]:
    for i in range(5):
        yield from range(i)
```

**3. Stacked Generator Pipeline**

```python
from collections.abc import Iterable, Iterator

def clean_data(source: Iterable[str]) -> Iterator[str]:
    # Pipeline stage 1: Lazily strip whitespace
    stripped = (row.strip() for row in source)

    # Pipeline stage 2: Lazily filter out empty lines
    non_empty = (row for row in stripped if row)

    # Pipeline stage 3: Lazily convert remaining lines
    return (row.lower() for row in non_empty)
```

**4. Structural Processing Pipeline with Generators**

```python
from collections.abc import Iterable, Iterator

# Stage 1: Structural change (lazy)
def row_merge(source: Iterable[list[str]]) -> Iterator[list[str]]:
    cluster: list[str] = []
    for row in source:
        if not row:
            continue
        if row and cluster:
            yield cluster
            cluster = row.copy()
        else:
            cluster.extend(row)
    if cluster:
        yield cluster

# Stage 2: Filtering (lazy)
def pass_valid(row: list[str]) -> bool:
    return row != "invalid"

# Stage 3: Stack the generators to create a pipeline
def process_logs(log_lines: Iterable[list[str]]) -> Iterator[list[str]]:
    merged_rows = row_merge(log_lines)
    valid_rows = filter(pass_valid, merged_rows)
    return (row for row in valid_rows)
```

**5. Recursive Generator with `yield from` and Pattern Matching**

```python
from typing import Any
from collections.abc import Iterator

def find_value(value: Any, node: dict | list | Any, path: list | None = None) -> Iterator[list]:
    if path is None:
        path = []

    match node:
        case dict() as dnode:
            for key in sorted(dnode.keys()):
                yield from find_value(value, node[key], path + [key])
        case list() as lnode:
            for index, item in enumerate(lnode):
                yield from find_value(value, item, path + [index])
        case _ as pnode:
            if pnode == value:
                yield path
```

**6. Functional Reduction**

```python
from functools import reduce
from collections.abc import Iterable
import operator

def compute_product(values: Iterable[int]) -> int:
    return reduce(operator.mul, values, 1)

def factorial(n: int) -> int:
    return compute_product(range(1, n + 1))
```
