---
name: python-configuration-observability
description: >
  Instructs the agent on cloud-native application configuration and observability.
  Covers environment variable configuration (Twelve-Factor methodology), routing logs
  to stdout, structured JSON logging, hierarchical loggers, cross-boundary request
  tracing with ContextVar, and framework-independent observability. Use when
  configuring applications, implementing logging, or adding tracing.
---

# Python Configuration & Observability Guidelines

You are an expert Python developer specializing in cloud-native application design. When asked to configure applications, implement logging, add tracing, or set up monitoring, you must follow cloud-native best practices and adhere to the following rules:

---

## Part 1: Configuration

### 1. Configuration in Environment Variables (Primary)

The code should be unique, but it must be adjustable through configuration to allow the same code to be deployed in different environments.

*   Store all configuration for the system in environment variables rather than using environment-specific files in the codebase.
*   **Operational configuration:** Parameters that connect parts of the system or relate to monitoring (e.g., database URLs, credentials, logging levels) must be strictly loaded from the environment.
*   **Feature configuration:** Parameters that change external behavior, like feature flags or theming, should also be loaded from the environment to allow business releases without code changes.
*   Fail loudly if a required operational environment variable is missing, rather than falling back to a default parameter.

### 2. TOML for Complex Configuration (Secondary)

When an application requires deeply nested configuration that cannot be cleanly expressed as flat environment variables:

*   Use the built-in `tomllib` module (Python 3.11+) to parse `.toml` documents.
*   When using `tomllib.load()`, open the configuration file in binary mode (`'rb'`).
*   Environment variables always take precedence over file-based configuration.

### 3. Python Namespace Classes for Configuration Defaults

When configuration requires complex defaults or conditional logic based on deployment environment:

*   Define a base configuration class with class-level attributes for default settings.
*   Use inheritance to define environment-specific overrides (e.g., `class Production(Configuration):`).
*   Always allow environment variables to override class-level defaults.

### 4. Stateless Processes

To allow a system to be horizontally scalable, it must be completely stateless.

*   Ensure that each process shares nothing and that any data that needs to persist is stored in a stateful backing service, such as a database, a cache (like Redis), or an object storage service.
*   Do not store state in the local hard drive or local memory between requests. The next request will likely be served by a different node.

---

## Part 2: Logging

### 5. Route Logs to `stdout`

Logs must be treated as an event stream.

*   The application process must not attempt to manage log files, rotate logs, or write them to the local hard drive.
*   Direct all logs to standard output (`stdout`) without any intermediate steps.
*   The execution environment (like a container orchestrator) will capture the `stdout` stream and route it to centralized logging facilities.

### 6. Strictly Use `logging` over `print()`

Do not lump application outputs into `print()` statements.

*   Use the `logging` module to direct all control information, audit summaries, and error messages.
*   Name loggers hierarchically based on their purpose or class, such as `logging.getLogger(f"error.{self.__class__.__name__}")` or `logging.getLogger(f"audit.input")`.
*   Use appropriate severity levels: `INFO` for happy-path control/audit messages, `WARNING` for deprecations, and `ERROR` for processing failures.

### 7. Structured JSON Logging

Implement logging in a way that supports programmatic analysis without contaminating business logic.

*   Configure structured JSON logging formatting strictly in the outermost Frameworks and Drivers layer.
*   When logging from inner layers, provide structured business data using the standard `extra` parameter with a dedicated namespace (e.g., `logger.info("Message", extra={"context": {"id": 123}})`). The outer layer's infrastructure formatter will extract and serialize this cleanly.

### 8. Configure Logging via `dictConfig`

For applications that need complex routing of log messages:

*   Define loggers, handlers, and formatters inside a configuration dictionary (which can be loaded from a TOML file).
*   Pass the dictionary to `logging.config.dictConfig()` to consistently route different loggers to their respective destinations.
*   Ensure the final handler streams to `stdout` in production environments.

---

## Part 3: Observability

### 9. Avoid Framework Coupling (The Dependency Rule)

Do not tightly couple business operations with framework-specific logging mechanisms.

*   Never pass or use framework-specific internal loggers inside the Domain or Application layers. This creates an outward dependency that violates the Clean Architecture Dependency Rule.
*   Always instantiate and use Python's standard `logging.getLogger(__name__)` in core business logic.
*   Keep framework-specific HTTP access logging completely separate from business operation application logs.

### 10. Cross-Boundary Request Tracing with `ContextVar`

To track operations as they flow across multiple architectural boundaries (e.g., from an initial web request, through controllers, into use cases and repositories), implement a unique trace ID.

*   Use Python's `contextvars.ContextVar` to hold the trace ID. This provides storage that functions correctly across both threaded and asynchronous boundaries.
*   Generate and set the trace ID at the system boundary before the request hits the controller (e.g., using FastAPI middleware).
*   Ensure the custom logging formatter extracts this `ContextVar` and automatically injects the trace ID into every formatted JSON log message.

---

## Code Examples

**1. Configuration via Environment Variables**

```python
import os

# Operational Configuration: Fails loudly with a KeyError if missing
DATABASE_URL = os.environ['DATABASE_URL']
SECRET_KEY = os.environ['SECRET_KEY']

# Feature Configuration: Safe extraction with a default value
ENABLE_NEW_UI = os.environ.get('ENABLE_NEW_UI', 'false').lower() == 'true'
```

**2. TOML Fallback with Environment Override**

```python
import os
import tomllib
from pathlib import Path
from typing import Any

def load_config(config_path: Path) -> dict[str, Any]:
    with config_path.open('rb') as f:
        file_config = tomllib.load(f)

    # Environment variables always take precedence
    file_config["database_url"] = os.environ.get(
        "DATABASE_URL", file_config.get("database_url", "")
    )
    return file_config
```

**3. Python Classes as Configuration Namespaces**

```python
import os

class Configuration:
    """Generic Configuration with default settings."""
    base_url = "https://api.example.com/v1"
    query_limit = 100
    timeout_seconds = 30

class Production(Configuration):
    """Production overrides using inheritance."""
    base_url = os.environ.get("API_BASE_URL", "https://api.example.com/v2")
    query_limit = 5000
```

**4. Cross-Boundary Tracing (`infrastructure/logging/trace.py`)**

```python
from contextvars import ContextVar
from typing import Optional
from uuid import uuid4

# Async-safe context variable to hold the trace ID
trace_id_var: ContextVar[Optional[str]] = ContextVar("trace_id", default=None)

def set_trace_id(trace_id: Optional[str] = None) -> str:
    """Set trace ID for current context at the framework boundary."""
    new_id = trace_id or str(uuid4())
    trace_id_var.set(new_id)
    return new_id

def get_trace_id() -> str:
    """Get current trace ID or generate a new one if missing."""
    current = trace_id_var.get()
    if current is None:
        current = str(uuid4())
        trace_id_var.set(current)
    return current
```

**5. Infrastructure JSON Formatter Streaming to stdout**

```python
import sys
import json
import logging
from datetime import datetime, timezone

class JsonFormatter(logging.Formatter):
    """Formats log records as JSON, incorporating Trace IDs and Context."""

    def format(self, record: logging.LogRecord) -> str:
        log_data = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "trace_id": get_trace_id(),
        }
        context = getattr(record, "context", None)
        if context:
            log_data["context"] = context
        return json.dumps(log_data)

def setup_logging() -> None:
    handler = logging.StreamHandler(stream=sys.stdout)
    handler.setFormatter(JsonFormatter())
    root = logging.getLogger()
    root.addHandler(handler)
    root.setLevel(logging.INFO)
```

**6. FastAPI Middleware for Request Tracing**

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def trace_requests(request: Request, call_next):
    incoming_trace = request.headers.get("X-Trace-ID")
    trace_id = set_trace_id(incoming_trace)
    response = await call_next(request)
    response.headers["X-Trace-ID"] = trace_id
    return response
```

**7. Framework-Independent Core Logging (application layer)**

```python
import logging

# Standard python logger; knows nothing about JSON, FastAPI, or ContextVars
logger = logging.getLogger(__name__)

async def create_task(title: str, project_id: int) -> None:
    # Logs cleanly; tracing ID is implicitly attached by the JSON Formatter
    logger.info(
        "Creating new task",
        extra={"context": {"title": title, "project_id": project_id}}
    )
    # ... business logic ...
```

**8. Hierarchical Logging via dictConfig**

```python
import logging
import logging.config

LOGGING_CONFIG = {
    "version": 1,
    "formatters": {
        "json": {
            "()": "myapp.logging.JsonFormatter",
        }
    },
    "handlers": {
        "stdout": {
            "class": "logging.StreamHandler",
            "stream": "ext://sys.stdout",
            "formatter": "json",
        }
    },
    "loggers": {
        "error": {"level": "ERROR", "handlers": ["stdout"]},
        "audit": {"level": "INFO", "handlers": ["stdout"]},
    },
    "root": {
        "level": "INFO",
        "handlers": ["stdout"],
    },
}

logging.config.dictConfig(LOGGING_CONFIG)
```
