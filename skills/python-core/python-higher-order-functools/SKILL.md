---
name: python-higher-order-functools
description: >
  Instructs the agent on using Python's built-in higher-order functions (map, filter,
  max, min) with lambda forms or pure functions, and the functools module for memoization
  (@cache, @lru_cache), currying (partial), reductions (reduce), and type-based
  polymorphism (@singledispatch). Use when processing collections, optimizing functions,
  or implementing functional design patterns.
---

# Python Higher-Order Functions & Functools Guidelines

You are an expert Python developer specializing in functional programming design patterns. When asked to process collections, optimize functions, bind arguments, reduce data, or handle multiple input types, you must utilize Python's built-in higher-order functions and the `functools` standard library. Adhere to the following rules:

---

## 1. Avoid Stateful For Loops

When transforming or selecting data from an iterable, do not initialize an empty list and use a `for` loop to append items.

*   Use `map()` to apply a scalar function to each item in a collection.
*   Use `filter()` to pass or reject items based on a boolean predicate function.

## 2. Using `max()` and `min()` as Higher-Order Functions

The `max()` and `min()` functions are reductions that act as higher-order functions when dealing with complex data structures.

*   Always use the `key=` keyword argument to inject a function that extracts the comparison value.
*   Do not manually unwrap, sort, and re-wrap data just to find minimum or maximum elements.

## 3. Lambdas vs. Pure Functions

Higher-order functions require a function as an argument.

*   **Lambdas:** Use `lambda` forms for short, single-expression functions used only once (e.g., `key=lambda x: x[1]`).
*   **Pure Functions:** For anything requiring type hints, complex logic, or reuse, define a pure function using `def`.
*   **PEP 8 Compliance:** Never assign a lambda object to a variable. If a function needs a name, use `def` or tools like `operator.itemgetter`.

## 4. Memoization with `@cache` and `@lru_cache`

Use memoization to drastically improve the performance of pure functions that frequently recalculate the same values (such as recursive mathematical algorithms).

*   Apply `@cache` for unbounded memoization.
*   Apply `@lru_cache(maxsize)` to bound the memory footprint. The cache discards the Least Recently Used items once `maxsize` is reached.
*   **Rule:** Only apply these decorators to pure functions free of side-effects. Applying them to functions with side effects will cause those side effects to be skipped upon cache hits.
*   Use `.cache_clear()` on the decorated function to manually invalidate the cache (e.g., in tests or when the underlying data resets).

## 5. Partial Application with `partial()`

Use `partial()` to create new, specialized functions from existing functions by pre-binding a subset of the required argument values.

*   This provides currying-like behavior, allowing you to simplify a complex function signature before passing it into higher-order functions like `map()` or `filter()`.
*   Positional arguments are bound in a strict left-to-right order. Keyword arguments can also be pre-bound.

## 6. General Reductions with `reduce()`

When built-in reductions like `sum()`, `len()`, or `max()` are insufficient, use `reduce()` to fold a binary operation across an iterable from left to right.

*   **Crucial Initialization Rule:** Always provide a proper `initial` value as the third argument to `reduce()`. If the source iterable is empty and no initial value is provided, it will raise an error.

## 7. Type-Based Polymorphism with `@singledispatch`

Avoid writing complex `if isinstance(...)` or `match` statements inside a function to handle different incoming data types.

*   Use `@singledispatch` to define a default, base function (which typically raises a `NotImplementedError`).
*   Use `@base_function_name.register` to implement overloaded, type-specific variations.
*   Name the overloaded functions `_` so they are bundled into the primary dispatcher.

---

## Code Examples

**1. Transforming Data with `map()` and Filtering with `filter()`**

```python
from collections.abc import Iterable, Iterator

# BAD: Imperative stateful loop
def convert_imperative(data: list[str]) -> list[float]:
    results = []
    for item in data:
        results.append(float(item))
    return results

# GOOD: Stateless functional mapping
def convert_functional(data: Iterable[str]) -> Iterator[float]:
    return map(float, data)

# GOOD: Higher-order filtering using a lambda predicate
def get_multiples(limit: int) -> Iterator[int]:
    return filter(lambda x: x % 3 == 0 or x % 5 == 0, range(limit))
```

**2. Custom Extrema with `max()` and `min()`**

```python
trip_data = [
    ((37.549, -76.330), (37.840, -76.273), 17.7246),
    ((37.840, -76.273), (38.331, -76.459), 30.7382),
    ((36.843, -76.298), (37.549, -76.331), 42.3962)
]

def get_longest_leg(trip: list[tuple]) -> tuple:
    return max(trip, key=lambda leg: leg[2])

def get_shortest_leg(trip: list[tuple]) -> tuple:
    return min(trip, key=lambda leg: leg[2])
```

**3. Memoization (`@lru_cache`)**

```python
from functools import lru_cache

@lru_cache(128)
def fibc(n: int) -> int:
    if n == 0: return 0
    if n == 1: return 1
    return fibc(n - 1) + fibc(n - 2)
```

**4. Partial Application (`partial()`)**

```python
from functools import partial

# Creates a new function that acts as pow(2, y)
exp2 = partial(pow, 2)

def test_exp2():
    assert exp2(12) == 4096
```

**5. General Reductions (`reduce()`)**

```python
from functools import reduce
from collections.abc import Iterable

def sum_of_squares(data: Iterable[float]) -> float:
    # Always supply the initial value to prevent errors with empty iterables
    return reduce(lambda x, y: x + y**2, data, 0.0)
```

**6. Type-Based Polymorphism (`@singledispatch`)**

```python
from functools import singledispatch
from typing import Any

@singledispatch
def zip_format(zip_code: Any) -> str:
    raise NotImplementedError(f"unsupported {type(zip_code)} for zip_format()")

@zip_format.register
def _(zip_code: int) -> str:
    return f"{zip_code:05d}"

@zip_format.register
def _(zip_code: str) -> str:
    if "-" in zip_code:
        zip_code, _ = zip_code.split("-")
    return f"{zip_code:0>5s}"
```
