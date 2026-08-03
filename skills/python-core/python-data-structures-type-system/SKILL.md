---
name: python-data-structures-type-system
description: >
  Instructs the agent on choosing the right built-in collections (mutable vs. immutable),
  mandating the typing module for static analysis via `ty`, enforcing Pydantic for
  run-time data validation, and using Union/Optional syntax, type aliases, NewType,
  Sequence, and Literal to enforce architectural boundaries. Use when designing data
  models, defining interfaces, or working with type annotations.
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

### 2. Mandatory Type Hints, Checked with `ty`

Type hints are essential to good software design; they prevent problems by enforcing rigor and formality without adding run-time overhead. Provide type hints for all collections, function parameters, and return values. `ty` is this repository's preferred static type checker (see the `python` master skill) — type hints are written for `ty` to verify, not left as unchecked decoration.

*   Use standard generics like `list[int]`, `set[str]`, or `dict[str, int]`.
*   For heterogeneous collections, define a union of types using the `|` operator (e.g., `list[int | float]`).
*   Decompose complex nested type hints into layers using `TypeAlias` to make them readable.
*   Use `typing.TypedDict` or `typing.NamedTuple` for dictionary-like or tuple-like structures with specific, named fields.
*   Run `uv run ty check` locally and in CI — see python-project-setup for wiring `ty` into pre-commit hooks and GitHub Actions.

### 2a. Union and Optional Types

Use the modern `|` syntax for union and optional types (Python 3.10+ syntax; this repo targets 3.11+ — see the `python` master skill) instead of `typing.Union`/`typing.Optional`.

*   A value that can be one of several types: `data: str | int`, not `Union[str, int]`.
*   An optional value (may be `None`): `user_id: int | None = None`, not `Optional[int]`.
*   At architectural boundaries (use-case inputs, function return values crossing a layer), use unions to make every valid shape of the data explicit rather than smuggling multiple meanings into one loosely-typed parameter.
*   Prefer a narrower union or a dedicated type over `X | None` when `None` is being used to represent something more specific than "absent" (e.g., use a `Literal["not_found"]` or a small result type instead of overloading `None` to mean several different failure states).

### 2b. Type Aliases for Readability, Not Just Nesting

Reach for `TypeAlias` (or the plain `NameX = ...` assignment form, which `ty` also recognizes) whenever a type signature is reused across the codebase or is complex enough that its shape isn't obvious at a glance — not only when a type is deeply nested.

```python
from typing import TypeAlias

UserId: TypeAlias = int
UserRecord: TypeAlias = dict[str, str]
UserList: TypeAlias = list[UserRecord]

def process_users(users: UserList) -> None: ...
```

*   A named alias documents intent (`UserList` reads as "a list of user records," not just "a list of dicts") — treat this as documentation that `ty` also verifies, not merely a readability nicety.
*   Prefer `NewType` over a plain alias specifically when two aliases share the same underlying type but must never be interchanged (see rule 4) — a plain `TypeAlias` does not prevent mixing `UserId` and `ProductId` if both are really just `int`.

### 2c. `Any` Is a Last Resort, Not a Convenience

Treat `typing.Any` as an explicit admission that a type is genuinely unknown or highly variable (e.g., data crossing a boundary with an external system that provides no schema) — not as a way to avoid writing a real type.

*   Within this codebase's own layers, encountering `Any` should prompt refactoring toward a specific type, a union, or a `TypeAlias` — never leave it as the default when the actual shape of the data is knowable.
*   Reserve `Any` for genuine external-system boundaries, and narrow it to a real type as soon as possible after the data enters the codebase (e.g., validate untyped external input into a Pydantic model immediately, per rule 7, rather than passing `Any` deeper into the call stack).

### 3. Prefer `Callable` Type Aliases over `Protocol` for Single-Operation Dependencies

For a single swappable operation (a "port" a use case depends on — sending a notification, looking up a price, persisting an entity), type it as a `Callable[[...], ...]` alias and pass a plain function, not a `Protocol`-typed single-method interface implemented by a class. See python-clean-architecture-dependency-inversion for the full pattern and rationale — this is this repo's default mechanism for dependency inversion, and it requires no class at all.

*   Reserve `typing.Protocol` for the narrower case of typing a *bundle* of several related callables structurally (e.g., a repository needing both a lookup and a save operation) when passing each function as a separate parameter becomes unwieldy — and even then, a `NamedTuple`/frozen `dataclass` of callables is usually simpler and is the preferred default; reach for `Protocol` only when callers at a public boundary need to pass "anything with this shape" without importing a specific bundle type.
*   Never use `Protocol` (or `ABC`) to type something that has real business-logic behavior beyond plain data or callables — if reference material shows a `Protocol` with one method implemented by classes with no other purpose, reformulate it as a `Callable` alias and plain functions.

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

**2. `Callable` Alias for a Single-Operation Dependency (preferred over `Protocol` + class)**

```python
from typing import Callable

# The "port" — a function type, not a class-based interface
NotifierFn = Callable[[str], None]

def send_email_notification(message: str) -> None:
    print(f"Sending email: {message}")

def notify(notifier: NotifierFn, message: str) -> None:
    notifier(message)

notify(send_email_notification, "Hello via email")
```

**2a. `Protocol` Reserved for a Bundle of Callables (narrow, less common case)**

```python
from typing import Protocol

class UserRepo(Protocol):
    get_by_id: Callable[[str], dict]
    save: Callable[[dict], None]
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

## Related guidance

For the full functional-lite dependency-inversion pattern this skill's `Callable`/`Protocol` guidance is part of (never `ABC`, `NamedTuple`/dataclass bundling, parameter-passing instead of constructor injection), see the `python-clean-architecture` package, specifically **python-clean-architecture-dependency-inversion**.
