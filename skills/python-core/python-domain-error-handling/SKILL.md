---
name: python-domain-error-handling
description: Instructs the agent on domain-level error handling, including generic Result[T] type patterns (a Success/Failure tagged union, not a single Any-typed class), custom exception hierarchies, concealing root causes, and architectural error propagation. Use cases that consume Result are plain functions with injected dependencies as parameters, never "interactor" classes with an execute method.
---

# Python Domain-Level Error Handling Guidelines

You are an expert Python developer specializing in Clean Architecture and robust error handling. When asked to implement error handling, validation, or cross-layer data flows, you must adhere to the following rules:

## 1. The Result Type Pattern (Application Layer)
Do not rely solely on exceptions for expected business logic control flow.
*   Implement `Result[T]` as a **generic tagged union** — `Success[T] | Failure` — never as a single dataclass with a loosely-typed `value: Any` field and an `Optional[Error]` field. A single class with both fields optional makes an invalid state representable (both set, or neither) and forces every consumer to check `is_success` before the type checker can narrow `value`'s type. A tagged union makes the invalid state unconstructable and lets `ty`/structural pattern matching narrow the type automatically — see python-pattern-matching.
*   This pattern makes success and failure paths explicit in function signatures and provides consistent error handling across the application.
*   Use standard error objects with specific error codes (e.g., `ErrorCode.NOT_FOUND` or `ErrorCode.VALIDATION_ERROR`).
*   Use cases that return a `Result` are plain functions receiving their dependencies as parameters (see the `python-clean-architecture` package, especially python-clean-architecture-dependency-inversion and python-clean-architecture-use-cases) — never a frozen-dataclass "interactor" class with dependency fields and a single `execute` method. That shape is a class wearing a function's clothing; drop the class and call the function directly.

## 2. Custom Exception Hierarchies & Concealment
When exceptions are necessary for domain constraints, insulate users from low-level implementation details.
*   Define custom, application-specific exception classes by inheriting from `Exception`.
*   When catching a low-level error (like `AttributeError` or `OSError`) to translate it into your custom domain exception, use the `raise CustomError(...) from None` syntax. This conceals the original, irrelevant traceback from leaking to the consumer.

## 3. Strict Error Propagation and Boundaries
Exceptions must not leak their implementation details across architectural boundaries.
*   **Application Layer:** Use cases must explicitly catch specific domain exceptions (like `ProjectNotFoundError`) and translate them into a `Failure(...)` value.
*   **Avoid Bare Excepts:** Never use a bare `except:` or `except Exception:` clause in the business logic to swallow unexpected errors. Let global, unexpected errors bubble up to the outer framework layers for 500-level logging.
*   **Interface Adapters:** When translating the `Result` for the web or CLI, map it to an `OperationResult` to explicitly handle success and failure formats without coupling the core logic to a web framework.

## 4. Functional Validation with Monads (Alternative)
For strict functional data pipelines, avoid `try/except` blocks that interrupt execution flow.
*   Use the `Either` monad (with `Left` and `Right` subclasses).
*   Return valid data wrapped in `Right` and error messages identifying invalid data wrapped in `Left`. 

---

## Code Examples

Below are best-practice examples demonstrating these domain-level error-handling strategies.

**1. Concealing Root Causes with Custom Exceptions**
```python
class DomainValidationError(Exception):
    """Custom exception for domain-specific validation failures."""
    pass

def parse_business_rule(data: dict) -> str:
    try:
        # A low-level technical error might occur here
        return data["deep"]["nested"]["key"].upper()
    except (KeyError, AttributeError):
        # Conceal the low-level cause from the outer layers using `from None`
        raise DomainValidationError("Invalid data structure provided.") from None
```

**2. The Result Type Pattern — Generic Tagged Union**
```python
from enum import Enum
from dataclasses import dataclass
from typing import Generic, TypeVar, Self

T = TypeVar("T")

class ErrorCode(Enum):
    NOT_FOUND = "NOT_FOUND"
    VALIDATION_ERROR = "VALIDATION_ERROR"

@dataclass(frozen=True)
class Error:
    code: ErrorCode
    message: str

    @classmethod
    def not_found(cls, entity: str, entity_id: str) -> Self:
        return cls(code=ErrorCode.NOT_FOUND, message=f"{entity} {entity_id} not found")

@dataclass(frozen=True)
class Success(Generic[T]):
    value: T

@dataclass(frozen=True)
class Failure:
    error: Error

Result = Success[T] | Failure  # a TypeAlias — see python-data-structures-type-system
```

Constructing `Success("done")` or `Failure(Error.not_found(...))` directly (no `Result.success()`/`Result.failure()` classmethod wrappers needed) makes the two cases distinct types the checker can narrow between — there is no state where a caller has to ask "is this actually a success" the way `is_success` did; the type itself already says so.

**3. Consuming a `Result` — Pattern Matching, Not `is_success`**
```python
def render(result: Result[dict]) -> dict:
    match result:
        case Success(value):
            return {"status": "ok", "data": value}
        case Failure(error):
            return {"status": "error", "code": error.code.value, "message": error.message}
```

**4. Translating Exceptions at the Application Boundary — a Function, Not an Interactor Class**
```python
def complete_project(
    get_project: Callable[[str], Project],
    save_project: Callable[[Project], None],
    project_id: str,
) -> Result[dict]:
    try:
        # 1. Standard Domain Operation
        project = get_project(project_id)
        completed = mark_project_completed(project)  # pure entity transition, see python-clean-architecture-entity-invariants
        save_project(completed)

        # 2. Return explicit success
        return Success({"id": project_id, "status": "COMPLETED"})

    # 3. Explicitly catch domain errors and translate to Failure(...)
    except ProjectNotFoundError:
        return Failure(Error.not_found("Project", project_id))
    except DomainValidationError as e:
        return Failure(Error(ErrorCode.VALIDATION_ERROR, str(e)))

    # NOTE: No generic `except Exception:` here.
    # Unexpected systemic errors bubble up to the framework/infrastructure layer.
```
The dependencies (`get_project`, `save_project`) are plain `Callable` parameters, not fields on a
frozen-dataclass "use case" class with an `execute` method — calling `complete_project(get_project_from_db, save_project_to_db, "abc123")` directly *is* the dependency injection; no class needs to be instantiated first. See python-clean-architecture-use-cases for the full pattern, including multi-step orchestration.