---
name: python-immutable-data-design
description: >
  Instructs the agent on designing immutable data structures and safe resource management.
  Covers typing.NamedTuple, frozen dataclasses, optional pyrsistent usage, context managers,
  and the benefits and trade-offs of immutability. Use when designing data containers,
  value objects, or managing external resources.
---

# Python Immutable Data Design Guidelines

You are an expert Python developer specializing in robust, stateless data design and safe resource management. When asked to design classes, data containers, or handle external resources like files and network connections, consider the following guidelines:

---

## 1. Prefer Immutability Where It Helps

During design, if an object is a relatively passive container of data and its attributes never change, prefer defining it as an immutable object rather than a standard stateful class. However, be pragmatic — SQLAlchemy models and async state are inherently mutable, so do not force immutability where it creates friction.

**Benefits of immutability:**
*   Eliminates a class of mutation bugs (accidental state changes).
*   Thread-safe and async-safe by default (no race conditions on reads).
*   Easier to reason about in functional pipelines.

**Trade-offs:**
*   "Updates" require creating new instances (copy-on-write), which can be verbose.
*   Not always compatible with ORMs or mutable framework APIs.
*   Can add overhead for objects that are frequently modified.

## 2. Using `typing.NamedTuple`

Use `NamedTuple` as a base class for simple, stateless value objects.

*   Attempting to change an instance attribute will safely raise an `AttributeError`.
*   If a complex initialization or an eagerly computed derived value is required, encapsulate it inside a `@classmethod` factory method instead of mutating after creation.

## 3. Using Frozen `@dataclass`

For more complex field definitions or customized initialization, use the `@dataclass` decorator with `frozen=True`.

*   To allow instances to be sorted, also include `order=True`.
*   Any attempt to change the state will safely raise a `dataclasses.FrozenInstanceError`.
*   To optimize memory usage (since dataclasses can use more memory than NamedTuples), consider adding `slots=True` to the decorator.

## 4. Optional: `pyrsistent` for Evolving State

When an application requires an immutable object that seems to need frequent "updates", the third-party `pyrsistent` library offers immutable collections and records with built-in evolution support. This is **opt-in** — use it when the benefits are clear, not as a blanket requirement.

*   Use `PRecord`, `PMap`, or `PVector` for immutable collections and records.
*   **Evolution, not Mutation:** Use pyrsistent's `.set()` method to clone the original object and create a new immutable instance with the changed value.

## 5. Mandatory Use of Context Managers (`with` statement)

When your code is entangled with external resources (such as disk files or network connections), guarantee that the resource is properly released to prevent memory and resource leaks.

*   Always wrap file operations and network connections in a `with` statement.
*   The context manager ensures that files are automatically and safely closed at the end of the indented block, even if an exception is raised during processing.

## 6. Creating Custom Context Managers

When you need to define a custom scope for an external resource or complex computation:

*   Implement the `__enter__()` method to start the context and return the usable object.
*   Implement the `__exit__()` method to explicitly release resources and handle cleanup.

---

## Code Examples

**1. Immutable NamedTuple with Factory Classmethod**

```python
from typing import NamedTuple
import math

class Point(NamedTuple):
    latitude: float
    longitude: float

class LegNT(NamedTuple):
    start: Point
    end: Point
    distance: float

    @classmethod
    def create(cls, start: Point, end: Point) -> "LegNT":
        dist = math.hypot(start.latitude - end.latitude, start.longitude - end.longitude)
        return cls(start=start, end=end, distance=round(dist, 4))
```

**2. Frozen Dataclass with Memory Optimization**

```python
from dataclasses import dataclass

# frozen=True ensures immutability, slots=True optimizes memory footprint
@dataclass(frozen=True, order=True, slots=True)
class PointDC:
    latitude: float
    longitude: float

@dataclass(frozen=True, slots=True)
class LegDC:
    start: PointDC
    end: PointDC
    distance: float
```

**3. Optional: pyrsistent for Evolving State**

```python
from pyrsistent import pmap, PRecord, field

# Using PMap: dictionary that evolves statelessly
def update_world_value():
    v = pmap({"hello": 42, "world": 3.14159})
    v2 = v.set("another", 2.71828)  # Returns a new immutable map
    return v2

# PRecord: complex record with strict typing
class PointPR(PRecord):
    latitude = field(type=float)
    longitude = field(type=float)
```

**4. Mandatory Context Managers for Safe I/O**

```python
from pathlib import Path
import csv
from collections.abc import Iterable

def write_safe_csv(target_path: Path, data: Iterable[list[int]]) -> None:
    # The 'with' statement guarantees the file handle is closed
    with target_path.open('w', newline='') as target_file:
        writer = csv.writer(target_file)
        writer.writerow(['col1', 'col2', 'col3'])
        writer.writerows(data)
```

**5. Creating a Custom Context Manager**

```python
from types import TracebackType

class ManagedResource:
    def __init__(self, config_value: str) -> None:
        self.config = config_value

    def __enter__(self) -> "ManagedResource":
        print(f"Acquiring resource with {self.config}")
        return self

    def __exit__(
        self,
        exc_type: type[Exception] | None,
        exc_val: Exception | None,
        exc_tb: TracebackType | None
    ) -> bool | None:
        print("Safely releasing resource.")
        return None
```
