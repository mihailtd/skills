---
name: python-fastapi-routing-validation
description: Guides the agent on handling multiple routes modularly using the APIRouter class, validating request bodies using Pydantic models, and properly extracting path and query parameters in FastAPI.
---

# FastAPI Routing and Validation Guidelines

You are an expert Python developer specializing in FastAPI. When asked to build or refactor FastAPI routes, request bodies, or parameters, adhere to the following rules:

## 1. Modular Routing with `APIRouter`

Do not use the main `FastAPI()` instance to handle all routing operations, as this prevents modularity in extensive applications.

- Always use the `APIRouter` class from the `fastapi` package to handle path operations for multiple routes.
- Group related routes in separate files (e.g., `todo.py`) and instantiate `router = APIRouter()`.
- Make the routes visible by using the `include_router()` method on the main `FastAPI` instance in the main application file (e.g., `app.include_router(todo_router)`).

## 2. Validating Request Bodies with Pydantic

Always validate request bodies to sanitize data and reduce the risk of malicious attacks.

**Functional-lite note:** `class Todo(BaseModel)` is a legitimate framework-
mandated exception to this repo's "no OOP classes for business logic" rule
(same category as SQLAlchemy's `DeclarativeBase`) — FastAPI's request/
response validation is wired through Pydantic's `BaseModel` machinery, there
is no functional-lite substitute for it, and it should be used without
hesitation. The rule this exception does *not* cover: don't attach business
logic methods to a `BaseModel` beyond validators — a `BaseModel` is a
request/response DTO (see python-clean-architecture-request-response-models
and python-clean-architecture-fastapi-boundary if this project uses the
`python-clean-architecture` package), not a place for service-style methods.

- Create models by subclassing `BaseModel` from the `pydantic` library.
- Use these models as type hints for the request body objects in your route handlers.
- **Nested Models:** Use nested Pydantic models when complex JSON body structures are required.
- **Provide Examples:** Use `model_config = ConfigDict(json_schema_extra={...})` inside the Pydantic model to embed example payloads for Swagger/ReDoc automatic documentation.

## 3. Path and Query Parameters

- **Path Parameters:** When identifying specific resources via URLs (e.g., `/todo/{todo_id}`), define the parameter in the route string and as an argument in the handler function.
- **Path Validation:** Import the `Path` class from `fastapi` to distinguish path parameters, add documentation, and enforce validation (e.g., `todo_id: int = Path(..., title="The ID of the todo")`).
- **Query Parameters:** Use query parameters to filter requests. Define them as arguments in the route handler that _are not_ homonymous with path parameters. You can explicitly validate them using `Query()` from `fastapi` (e.g., `query: str = Query(None)`).

---

## Code Example

Below is a best-practice example demonstrating these concepts.

**models.py (Pydantic Models)**

```python
from pydantic import BaseModel, ConfigDict

class TodoItem(BaseModel):
    model_config = ConfigDict(
        json_schema_extra={
            "examples": [
                {"item": "Read the next chapter of the book"}
            ]
        }
    )

    item: str

class Todo(BaseModel):
    id: int
    item: str
```

**todo_routes.py (APIRouter, Path parameters, and Body Validation)**

```python
from fastapi import APIRouter, Path, Query
from models import Todo, TodoItem

todo_router = APIRouter()
todo_list = []

# Request Body Validation using the Todo model
@todo_router.post("/todo")
async def add_todo(todo: Todo) -> dict:
    todo_list.append(todo)
    return {"message": "Todo added successfully."}

# Path Parameter explicitly defined with Path()
@todo_router.put("/todo/{todo_id}")
async def update_todo(todo_data: TodoItem, todo_id: int = Path(..., title="The ID of the todo to be updated")) -> dict:
    for todo in todo_list:
        if todo.id == todo_id:
            todo.item = todo_data.item
            return {"message": "Todo updated successfully."}
    return {"message": "Todo with supplied ID doesn't exist."}
```

**main.py (Wiring it all together)**

```python
from fastapi import FastAPI
from todo_routes import todo_router

app = FastAPI()

# Make the modular routes visible
app.include_router(todo_router)
```

## Related guidance

For where Pydantic request/response models and route handlers sit relative to Clean Architecture's layers (if this project uses the `python-clean-architecture` package), see python-clean-architecture-fastapi-boundary. For update endpoints specifically, see python-fastapi-partial-updates — a naive full-object overwrite on a PUT/PATCH route silently clears any field the client didn't send.
