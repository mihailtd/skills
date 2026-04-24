---
name: python-functional-programming-recursion-tco

description: Instructs the agent on manually optimizing recursive algorithms into explicit for loops, generator expressions, or using explicit stacks to bypass Python's recursion limits and lack of automatic tail-call optimization.
---

# Python Recursion and Tail-Call Optimization Guidelines

You are an expert Python developer specializing in functional programming and performance optimization. When asked to write recursive algorithms or optimize recursive functions, you must adhere to the following rules:

## 1. Avoid Deep Recursion

Python imposes a strict recursion limit (default of 1,000) to prevent infinite loops, and it does not feature an optimizing compiler that automatically performs Tail-Call Optimization (TCO) [3-5].

- Do not rely on pure recursion for large datasets or deep call trees, as it will raise a `RecursionError` [6].
- Always optimize tail-recursive algorithms manually [7, 8].

## 2. Manual Tail-Call Optimization (Iteration)

When a function's final operation is a recursive call to itself (tail recursion), eliminate the overhead of the function call stack by refactoring the recursion into an explicit iteration [9].

- Replace the recursive return statement with an explicit `for` or `while` loop [10, 11].
- Maintain the state of the computation using explicit local variables that update on each iteration [10, 12].

## 3. Optimizing Collection Processing (Generators)

Do not build large, materialized lists using recursive concatenation [13].

- Optimize recursive collection processing by writing a generator function [11].
- Use a `for` loop and the `yield` statement to evaluate and emit values lazily, rather than accumulating them in a recursive stack [14, 15].

## 4. Explicit Stacks for Complex Recursion

For complex or multiple recursions (such as traversing an indefinite hierarchical directory tree) where simple tail-call optimization isn't feasible, manage the state manually [16].

- Use a `collections.deque` object as an explicit Last-In-First-Out (LIFO) stack to hold pending work [17].
- Push new tasks to the stack using `append()` and process them using `pop()` inside a `while` loop until the stack is empty [17, 18].

---

## Code Examples

Below are best-practice examples demonstrating how to manually optimize recursions in Python.

**1. Manual Tail-Call Optimization (Factorials)**

```python
# BAD: Pure recursion will fail for n >= 1000 due to stack limits [19, 20].
def fact(n: int) -> int:
    if n == 0:
        return 1
    else:
        return n * fact(n - 1)

# GOOD: Manually optimized using an explicit loop and state variable [10].
def facti(n: int) -> int:
    if n == 0:
        return 1
    f = 1
    for i in range(2, n + 1):
        f = f * i
    return f
2. Optimizing Collection Processing (Generators)
from collections.abc import Callable, Iterable, Iterator
from typing import TypeVar

DomT = TypeVar("DomT")
RngT = TypeVar("RngT")

# BAD: Recursively building a list consumes excessive memory and hits stack limits [13, 21].
def mapr(f: Callable[[DomT], RngT], collection: list[DomT]) -> list[RngT]:
    if len(collection) == 0:
        return []
    return mapr(f, collection[:-1]) + [f(collection[-1])]

# GOOD: Tail-call optimized into a generator function that yields lazily [14, 15].
def mapg(f: Callable[[DomT], RngT], C: Iterable[DomT]) -> Iterator[RngT]:
    for x in C:
        yield f(x)
3. Using an Explicit Stack (collections.deque)
from collections import deque
from pathlib import Path

# GOOD: Replaces tree-traversal recursion with an explicit deque stack [17, 18].
def all_print(start: Path) -> int:
    count = 0
    pending: deque[Path] = deque([start])

    while pending:
        dir_path = pending.pop()
        for path in dir_path.iterdir():
            if path.is_file() and path.suffix == '.py':
                count += path.read_text().count("print")
            elif path.is_dir() and not path.stem.startswith('.'):
                # Add subdirectory to the explicit stack instead of recursing
                pending.append(path)

    return count
```
