---
name: python-logging-in-practice
description: Instructs the agent on practical logging strategies, including action-oriented log levels, setting up request IDs and correlation via ContextVar, securing audit trails, and integrating with FastAPI middleware.
---

# Python Logging in Practice Guidelines

You are an expert Python developer specializing in system observability and API development. When asked to implement logging, trace requests, or configure application monitoring, you must adhere to the following practical rules:

## 1. Action-Oriented Log Level Strategy
Do not categorize log levels by vague definitions of "importance". Define them by the **action** the development or operations team must take [1, 2].
*   **`ERROR`**: Use for unrecoverable, unexpected exceptions (like 500-level HTTP responses). These indicate the application is broken and require developer review and bug-fixing [3-5]. Do *not* use `ERROR` for client-side mistakes (like 400 Bad Request); a customer sending malformed data is expected behavior and requires no developer action [5, 6].
*   **`WARNING`**: Use for automatically recovered errors (like external service retries) or deprecated usage. A sudden spike requires investigation, but individual logs do not [7].
*   **`INFO`**: Use to track the general flow and audit trails of the system (e.g., "Incoming request", "Payment processed") [7].
*   **`DEBUG`**: Use for highly detailed execution context. These should be suppressed in production to prevent log explosion and disk space issues [7-9].

## 2. Request IDs and Cross-Boundary Correlation
To trace a specific task across asynchronous operations and microservices, you must uniquely identify each request [10, 11].
*   Generate a unique Request ID (or Trace ID) at the system boundary if one is not provided in the `X-Request-ID` HTTP header [12, 13].
*   Store this ID using Python's `contextvars.ContextVar` [14, 15]. This provides thread-safe and async-safe storage without explicitly passing the ID through every function signature [14].
*   Ensure custom log formatters automatically extract this `ContextVar` to inject the Request ID into every log message [16].

## 3. FastAPI Middleware Integration
Centralize the generation of Request IDs and audit trails at the web framework level.
*   Use FastAPI's `@app.middleware("http")` decorator (or inherit from Starlette's `BaseHTTPMiddleware`) to intercept incoming requests [17-19].
*   Inside the middleware, extract/generate the Request ID, set the `ContextVar`, and log the incoming request context (e.g., `request.method`, `request.url.path`, `request.client.host`) [17].
*   Inject the Request ID back into the outgoing response headers for end-to-end traceability [20].

## 4. Secure Audit Trails
Logs must provide context to replicate a problem, but they must never expose sensitive data [21].
*   Always include relevant contextual identifiers (e.g., `user_id`, `transaction_id`) [22].
*   **Never** log Passwords, API keys, Personally Identifiable Information (PII), or full credit card numbers [9, 22]. Mask or hash these details if they must be referenced [22].

---

## Code Examples

Below are best-practice examples demonstrating how to implement actionable logging and correlation in FastAPI.

**1. Request ID Context Management (`trace.py`)**
```python
from contextvars import ContextVar
from uuid import uuid4

# Async-safe context variable to hold the Request ID
request_id_var: ContextVar[str] = ContextVar("request_id", default="-")

def get_request_id() -> str:
    return request_id_var.get()

def set_request_id(req_id: str | None = None) -> str:
    new_id = req_id or str(uuid4())
    request_id_var.set(new_id)
    return new_id
2. FastAPI Middleware Integration (middleware.py)
import logging
from fastapi import FastAPI, Request
from .trace import set_request_id

logger = logging.getLogger("api.middleware")
app = FastAPI()

@app.middleware("http")
async def request_tracking_middleware(request: Request, call_next):
    # 1. Extract existing header or generate a new Request ID
    req_id = request.headers.get("X-Request-ID")
    
    # 2. Bind the ID to the current async context
    current_req_id = set_request_id(req_id)
    
    # 3. Log the audit trail (with safe contextual data)
    logger.info(
        f"Incoming {request.method} {request.url.path} from {request.client.host}"
    )
    
    # 4. Process the request
    response = await call_next(request)
    
    # 5. Inject the Request ID into the response for client correlation
    response.headers["X-Request-ID"] = current_req_id
    return response
3. Action-Oriented Log Levels & Data Masking (services.py)
import logging
from .trace import get_request_id

logger = logging.getLogger("domain.billing")

async def process_subscription(user_id: int, credit_card: str):
    # GOOD: Audit trail with context, masking the PII
    masked_card = f"****-****-****-{credit_card[-4:]}"
    logger.info(f"[{get_request_id()}] Processing subscription for user={user_id}, card={masked_card}")
    
    try:
        # .. domain logic ..
        pass
        
    except NetworkTimeoutError:
        # WARNING: Automatically retryable/recoverable; track spikes, no immediate action
        logger.warning(f"[{get_request_id()}] Payment gateway timeout, retrying...")
        
    except InvalidCardError:
        # INFO: Client-side error (400 Bad Request). Business as usual, no developer fix needed.
        logger.info(f"[{get_request_id()}] Payment rejected: Invalid card details.")
        
    except DatabaseConnectionError:
        # ERROR: Unrecoverable system failure (500 Internal Error). Requires developer fix.
        logger.exception(f"[{get_request_id()}] Database crash during payment processing.")
```