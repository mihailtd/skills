---
name: python-domain-error-handling
description: Instructs the agent on domain-level error handling, including Result type patterns, custom exception hierarchies, concealing root causes, and architectural error propagation.
---

# Python Domain-Level Error Handling Guidelines

You are an expert Python developer specializing in Clean Architecture and robust error handling. When asked to implement error handling, validation, or cross-layer data flows, you must adhere to the following rules:

## 1. The Result Type Pattern (Application Layer)
Do not rely solely on exceptions for expected business logic control flow [3, 4]. 
*   Implement a `Result` class that encapsulates either a successful value or a standardized `Error` object [5].
*   This pattern makes success and failure paths explicit in function signatures and provides consistent error handling across the application [4].
*   Use standard error objects with specific error codes (e.g., `ErrorCode.NOT_FOUND` or `ErrorCode.VALIDATION_ERROR`) [4].

## 2. Custom Exception Hierarchies & Concealment
When exceptions are necessary for domain constraints, insulate users from low-level implementation details [6, 7].
*   Define custom, application-specific exception classes by inheriting from `Exception` [7].
*   When catching a low-level error (like `AttributeError` or `OSError`) to translate it into your custom domain exception, use the `raise CustomError(...) from None` syntax [8]. This conceals the original, irrelevant traceback from leaking to the consumer [9].

## 3. Strict Error Propagation and Boundaries
Exceptions must not leak their implementation details across architectural boundaries [10, 11].
*   **Application Layer:** Use cases must explicitly catch specific domain exceptions (like `ProjectNotFoundError`) and translate them into `Result.failure()` responses [12, 13]. 
*   **Avoid Bare Excepts:** Never use a bare `except:` or `except Exception:` clause in the business logic to swallow unexpected errors [14, 15]. Let global, unexpected errors bubble up to the outer framework layers for 500-level logging [11, 16].
*   **Interface Adapters:** When translating the `Result` for the web or CLI, map it to an `OperationResult` to explicitly handle success and failure formats without coupling the core logic to a web framework [17, 18].

## 4. Functional Validation with Monads (Alternative)
For strict functional data pipelines, avoid `try/except` blocks that interrupt execution flow.
*   Use the `Either` monad (with `Left` and `Right` subclasses) [19].
*   Return valid data wrapped in `Right` and error messages identifying invalid data wrapped in `Left` [19]. 

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
2. The Result Type Pattern
from enum import Enum
from dataclasses import dataclass
from typing import Any, Optional, Self

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
class Result:
    """Represents success or failure of a use case execution."""
    value: Any = None
    error: Optional[Error] = None

    @property
    def is_success(self) -> bool:
        return self.error is None

    @classmethod
    def success(cls, value: Any) -> Self:
        return cls(value=value)

    @classmethod
    def failure(cls, error: Error) -> Self:
        return cls(error=error)
3. Translating Exceptions at the Application Boundary
from dataclasses import dataclass

@dataclass(frozen=True)
class CompleteProjectUseCase:
    project_repository: ProjectRepository

    def execute(self, project_id: str) -> Result:
        try:
            # 1. Standard Domain Operation
            project = self.project_repository.get(project_id)
            project.mark_completed()
            self.project_repository.save(project)
            
            # 2. Return explicit success
            return Result.success({"id": project_id, "status": "COMPLETED"})
            
        # 3. Explicitly catch domain errors and translate to Result.failure()
        except ProjectNotFoundError:
            return Result.failure(Error.not_found("Project", project_id))
        except DomainValidationError as e:
            return Result.failure(Error(ErrorCode.VALIDATION_ERROR, str(e)))
        
        # NOTE: No generic `except Exception:` here. 
        # Unexpected systemic errors bubble up to the framework/infrastructure layer.
```