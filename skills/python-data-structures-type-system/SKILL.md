---
name: python-data-structures-type-system
description: >
  Instructs the agent on choosing the right built-in collections (mutable vs. immutable),
  mandating the typing module for static analysis via mypy, enforcing Pydantic for
  run-time data validation, and using Protocol, NewType, Sequence, and Literal to
  enforce architectural boundaries. Use when designing data models, defining interfaces,
  or working with type annotations.
---

# Python Data Structures & Type System Guidelines

You are an expert Python developer specializing in data structures, static type checking, architectural boundaries, and data validation. When asked to design collections, define interfaces, or model domain concepts, you must choose appropriate structures, apply comprehensive type hints, and leverage Python's type system. Adhere to the following rules:

---

## Part 1: Data Structures

### 1. Choosing the Right Data Structure

Before putting data into a collection, map the unique features of the collection to the problem.

*   **Sets (`set` / `frozenset`):** Use when focused on checking existence or membership of a value, and order does not matter. Duplicate values are ignored.
*   **Lists (`list`):** Use when you need to identify items by index position, preserve order, or allow duplicates.
*   **Tuples (`tuple`):** Use when the number of items in the collection is fixed (e.g., RGB colors always have three values).
*   **Dictionaries (`dict`):** Use when you need to identify items by a distinct key rather than a numerical index.
*   **Mutability Constraints:** `set` items and `dict` keys must be immutable objects (like numbers, strings, `tuple`, or `frozenset`). Never use a mutable `list`, `dict`, or `set` as a dictionary key or set item.

---

## Part 2: Static Type Hinting

### 2. Mandatory Type Hints (`mypy`)

Type hints are essential to good software design; they prevent problems by enforcing rigor and formality without adding run-time overhead. Provide type hints for all collections.

*   Use standard generics like `list[int]`, `set[str]`, or `dict[str, int]`.
*   For heterogeneous collections, define a union of types using the `|` operator (e.g., `list[int | float]`).
*   Decompose complex nested type hints into layers using `TypeAlias` to make them readable.
*   Use `typing.TypedDict` or `typing.NamedTuple` for dictionary-like or tuple-like structures with specific, named fields.

### 3. Duck-Typing Interfaces with `Protocol`

Avoid rigid class hierarchies and explicit inheritance where possible.

*   Use `typing.Protocol` to define structural subtyping interfaces. This combines Python's duck typing flexibility with static type checking benefits.
*   Classes implicitly conform to a Protocol just by implementing the required methods — no subclassing is needed.
*   This aligns with the Dependency Inversion Principle without forcing inner layers to inherit from abstract base classes.

### 4. Distinct Domain Identifiers with `NewType`

Avoid "primitive obsession" where domain identifiers (like User IDs or Product IDs) are passed around as raw integers or strings.

*   Use `typing.NewType` to create distinct types for different domain concepts.
*   Static type checkers will catch accidental mixing of similar but conceptually different values (e.g., passing a `ProductId` to a function expecting a `UserId`).

### 5. Flexible Collections with `Sequence`

When defining interfaces that accept collections across architectural boundaries, prioritize flexibility.

*   Use `typing.Sequence` instead of rigid types like `list` or `tuple`.
*   This allows the function to work with any sequence type without breaking the contract, satisfying the Liskov Substitution Principle and requiring only the minimal interface needed.

### 6. Boundary Constraints with `Literal`

When crossing architectural boundaries, ensure data strictly conforms to expected values.

*   Use `typing.Literal` to specify exact string or integer values a variable can take.
*   This enforces specific values directly at the interface boundaries, creating more precise contracts.

---

## Part 3: Run-time Validation

### 7. Run-time Validation with Pydantic

Python does no data type checks or value range checks at run-time based on standard type hints. If an application requires strict run-time checking or conversion of complex input data (like JSON or CSV rows), use the `pydantic` package.

*   Define complex objects by subclassing `pydantic.BaseModel`.
*   Use `BaseModel` to automatically convert source data (like parsing strings into `datetime` objects).
*   When narrowing domains (e.g., checking ranges or applying regex patterns), use the `Annotated` type hint combined with Pydantic's `Field` or validation functions.

---

## Code Examples

**1. Data Structure Selection and Static Type Hints**

```python
from typing import TypeAlias

# Tuples for fixed-size, immutable data
Coordinates = tuple[float, float]

# Lists for position-based, ordered collections
PathList: TypeAlias = list[Coordinates]

# Sets for fast membership testing
ValidCommands: set[str] = {"start", "stop", "pause"}

# Dictionary mapping distinct string keys to complex values
SystemPaths: dict[str, PathList] = {
    "route_a": [(36.12, -86.67), (36.14, -86.65)],
    "route_b": [(37.22, -85.12)]
}

def is_valid_command(cmd: str) -> bool:
    return cmd in ValidCommands
```

**2. Protocol for Duck-Typing Interfaces**

```python
from typing import Protocol

class NotificationPort(Protocol):
    def send_notification(self, message: str) -> None:
        ...

# Implicitly conforms — no subclassing required
class EmailNotifier:
    def send_notification(self, message: str) -> None:
        print(f"Sending email: {message}")

class NotificationService:
    def __init__(self, notifier: NotificationPort):
        self.notifier = notifier
```

**3. NewType for Domain Identifiers**

```python
from typing import NewType

UserId = NewType('UserId', int)
ProductId = NewType('ProductId', int)

def process_order(user_id: UserId, product_id: ProductId) -> None:
    print(f"Processing order for User {user_id} and Product {product_id}")

user = UserId(1)
product = ProductId(1)
process_order(user, product)
# Type checker flags this: process_order(product, user)
```

**4. Sequence and Literal for Boundary Constraints**

```python
from typing import Sequence, Literal

def calculate_total(items: Sequence[float]) -> float:
    return sum(items)

calculate_total([1.0, 2.0, 3.0])  # Works with list
calculate_total((4.0, 5.0, 6.0))  # Works with tuple

LogLevel = Literal["DEBUG", "INFO", "WARNING", "ERROR"]

def set_log_level(level: LogLevel) -> None:
    print(f"Setting log level to {level}")
```

**5. Strict Run-time Validation with Pydantic**

```python
import datetime
from enum import StrEnum
from typing import Annotated
from pydantic import BaseModel, Field

class LevelClass(StrEnum):
    DEBUG = "DEBUG"
    INFO = "INFO"
    WARNING = "WARNING"
    ERROR = "ERROR"

class LogData(BaseModel):
    date: datetime.datetime
    level: LevelClass
    module: Annotated[str, Field(pattern=r'^\w+$')]
    message: str

def parse_log_data(raw_data: dict) -> LogData:
    return LogData.model_validate(raw_data)
```
