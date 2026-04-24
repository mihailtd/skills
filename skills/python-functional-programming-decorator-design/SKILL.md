---
name: python-decorator-design
description: Instructs the agent on creating composite functions using decorators to cleanly handle cross-cutting concerns like null-awareness, bad data filtering, and logging, while strictly mandating the use of functools.wraps.
---

# Python Decorator Design Guidelines

You are an expert Python developer specializing in functional programming and decorator design. When asked to implement cross-cutting concerns (such as logging, data validation, or null-awareness) or to create composite functions, you must utilize Python decorators and adhere to the following rules:

## 1. Mandatory use of `functools.wraps`

When writing a wrapper function inside a decorator, you must always apply the `@wraps(function)` decorator from the `functools` module to the wrapper [4].

- Python doesn't inherently adjust the internal bytecode of a function when decorating; it defines a new function that wraps the original [5].
- Using `@wraps` ensures that the resulting decorated function retains the metadata (like `__name__` and `__doc__` attributes) of the original base function [4].

## 2. Isolate Cross-Cutting Concerns

Do not clutter core application functions with infrastructure-level boilerplate [6, 7].

- Use decorators to implement cross-cutting concerns like logging, auditing, security checks, and handling incomplete/bad data [7].
- Let the base function focus on the "happy path" (ideal processing), while the decorator intercepts arguments to sanitize data, or intercepts the return value to apply formatting [8, 9].

## 3. Implement Null-Awareness Gracefully

In datasets where missing values (e.g., `None`) are common, avoid writing `if value is None:` inside every mathematical or domain function [10, 11].

- Create a `@nullable` decorator that checks if the input is `None`.
- If it is `None`, immediately return `None`; otherwise, evaluate the base function [12, 13].

## 4. Handle Bad Data and Parameterized Decorators

When dealing with strings that need cleaning before conversion (e.g., removing currency symbols or commas), use decorators to trap exceptions and retry the operation [14].

- Use a `try...except` block in the wrapper. If a `ValueError` occurs, clean the data and attempt the base function again [14, 15].
- When your decorator needs configuration (like which characters to remove), create a parameterized decorator: an outer function that takes the arguments, returning an abstract decorator, which then wraps the base function [16-18].

---

## Code Examples

Below are best-practice examples demonstrating how to properly use decorators to create composite functions for cross-cutting concerns.

**1. Null-Awareness Decorator (with `wraps`)**

```python
from collections.abc import Callable
from functools import wraps

# Decorator to cleanly handle missing data without cluttering the math function
def nullable(function: Callable[[float], float]) -> Callable[[float | None], float | None]:
    @wraps(function)
    def null_wrapper(value: float | None) -> float | None:
        return None if value is None else function(value)
    return null_wrapper

@nullable
def st_miles(nm: float) -> float:
    # Base function focuses entirely on the "happy path" math
    return 1.15078 * nm
2. Parameterized Decorator for Bad Data Filtering
from collections.abc import Callable
from functools import wraps
from typing import Any, TypeVar

T = TypeVar('T')

# Outer function accepts the configuration parameters
def bad_char_remove(*bad_chars: str) -> Callable[[Callable[[str], T]], Callable[[str], T]]:

    # The concrete decorator
    def cr_decorator(function: Callable[[str], T]) -> Callable[[str], T]:

        # The wrapper function that handles the cross-cutting data cleaning concern
        @wraps(function)
        def wrap_char_remove(text: str, **kwargs: Any) -> T:
            try:
                # Attempt the happy path
                return function(text, **kwargs)
            except ValueError:
                # Intercept the failure, clean the data, and retry
                cleaned = text
                for char in bad_chars:
                    cleaned = cleaned.replace(char, "")
                return function(cleaned, **kwargs)

        return wrap_char_remove
    return cr_decorator

# Usage: Decorator strips $ and , before allowing the base int() or float() to run
@bad_char_remove("$", ",")
def currency_to_float(text: str, **kwargs: Any) -> float:
    return float(text, **kwargs)
```
