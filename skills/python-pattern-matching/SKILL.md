---
name: python-pattern-matching
description: Instructs the agent on implementing Python 3.10+ structural pattern matching using the match and case statements to cleanly process heterogeneous collections of data, avoiding imperative isinstance() chains.
---

# Python Structural Pattern Matching Guidelines

You are an expert Python developer specializing in modern Python 3.10+ features. When asked to process heterogeneous collections of data, unpack complex structures, or distinguish between multiple object types, you must utilize structural pattern matching and adhere to the following rules:

## 1. Avoid `isinstance()` Chains
When evaluating an object that could be one of several distinct types (a union of types), do not use a long, imperative series of `if isinstance(...)` and `elif isinstance(...)` statements [2-5]. 
*   Use the `match` statement paired with `case` clauses. 
*   This provides a much more readable, declarative way to implement distinct behaviors based on an object's type or structure [5].

## 2. The Class Pattern
To match objects of specific classes, use the class pattern syntax: `case ClassName() as variable_name:` [4].
*   While this syntax looks like object instantiation, it is actually an expression that checks if the object matches the class and binds it to the specified variable [6].
*   You can also match specific internal attributes structurally, such as `case Referenced(_, [Path()]) as single:` to match a `Referenced` object where the second parameter is a list containing exactly one `Path` instance [7].

## 3. Structural Unpacking
Leverage pattern matching to simultaneously identify data types and unpack their contents into variables [8, 9].
*   **Sequences:** Use list or tuple patterns (e.g., `case [float(lat), float(lon)]:`) to match a sequence of specific types and bind them directly to variables [8].
*   **Mappings:** Use dictionary patterns (e.g., `case {"lat": float(lat), "lon": float(lon)}:`) to cleanly extract values from a dictionary while verifying its keys [10].

## 4. Case Ordering and the Wildcard
The `match` statement evaluates `case` clauses strictly in order [11].
*   **Most Specific First:** You must place more-specific cases (e.g., a class with a specific internal list structure) before less-specific cases (e.g., matching the general class itself) [11].
*   **Wildcard Fallback:** Always include a `case _:` wildcard clause at the end of your `match` block to catch unexpected types [4, 5]. Typically, this block should raise a `ValueError` or `TypeError` to explicitly fail on astonishing inputs [4, 5].

---

## Code Examples

Below are best-practice examples demonstrating how to replace imperative type checking and unpacking with structural pattern matching.

**1. Replacing `isinstance()` with the Class Pattern**
```python
from collections.abc import Iterable
from pathlib import Path

# BAD: Imperative type checking
def process_files_imperative(source: Iterable[Unreferenced | Referenced]) -> None:
    for file in source:
        if isinstance(file, Unreferenced):
            print(f"delete {file}")
        elif isinstance(file, Referenced):
            print(f"keep {file}")
            
# GOOD: Structural pattern matching
def process_files_declarative(source: Iterable[Unreferenced | Referenced]) -> None:
    for file in source:
        match file:
            # Matches a specific internal structure first
            case Referenced(_, [Path()]) as single:
                print(f"Single use reference: {single}")
            # Matches the general class
            case Referenced() as ref:
                print(f"Multiple use reference: {ref}")
            case Unreferenced() as unref:
                print(f"delete {unref}")
            # Catch-all wildcard for safety
            case _:
                raise ValueError(f"unexpected type {type(file)}")
2. Type Conversion and Structural Unpacking
from ast import literal_eval

# GOOD: Cleanly parsing multiple diverse structures (lists, dicts, strings)
def parse_coordinates(item: list | dict | str) -> tuple[float, float]:
    match item:
        # Matches a list of two floats and unpacks them
        case [float(lat), float(lon)]:
            return lat, lon
            
        # Matches a dictionary with specific keys and unpacks the values
        case {"lat": float(lat), "lon": float(lon)}:
            return lat, lon
            
        # Matches a string and applies conversion
        case str(sll):
            return literal_eval(sll)
            
        case _:
            raise ValueError(f"unexpected types in {item!r}")
```