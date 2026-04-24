---
name: python-fastapi-response-error-handling
description: Instructs the agent on building customized response models using Pydantic, defining specific HTTP status codes, and handling errors gracefully using FastAPI's HTTPException.
---

# FastAPI Response Models and Error Handling Guidelines

You are an expert Python developer specializing in FastAPI. When asked to construct API responses, format outgoing data, or handle errors, you must adhere to the following rules:

## 1. Response Models

Response models serve as templates for returning data from an API route path. Use them to strictly define outgoing data formats and deliberately hide sensitive fields (such as internal database IDs or passwords).

- Always build response models by subclassing Pydantic's `BaseModel`.
- Declare the response model in the route decorator using the `response_model` argument (e.g., `@router.get("/items", response_model=ItemResponse)`).
- Do not manually strip fields in the route handler; let the `response_model` handle data filtering automatically.

## 2. HTTP Status Codes

Always return appropriate HTTP status codes for individual events rather than relying solely on the default `200 OK`.

- Import the `status` module from `fastapi` to use readable status code variables (e.g., `status.HTTP_201_CREATED`).
- For successful operations that create a resource (POST), override the default status code by adding the `status_code` argument to the route decorator (e.g., `@router.post("/items", status_code=status.HTTP_201_CREATED)`).

## 3. Error Handling with `HTTPException`

Do not return standard dictionaries or strings to indicate a failure or error.

- Always raise an `HTTPException` (imported from `fastapi`) to indicate a fault or issue in the request flow.
- The `HTTPException` class must take at least two arguments:
  - `status_code`: The appropriate HTTP error code (e.g., `status.HTTP_404_NOT_FOUND` or `status.HTTP_409_CONFLICT`).
  - `detail`: A clear, descriptive accompanying message to be sent to the client.

---

## Code Example

Below is a best-practice example demonstrating response models, custom status codes, and proper error handling.

**models.py (Defining the Response Model)**

```python
from pydantic import BaseModel
from typing import List

# Standard model (contains sensitive/internal data like ID)
class Todo(BaseModel):
    id: int
    item: str

# Response model (hides the internal ID to format outgoing data)
class TodoItemResponse(BaseModel):
    item: str

class TodoItemsResponse(BaseModel):
    todos: List[TodoItemResponse]
routes.py (Applying Response Models and Error Handling)
from fastapi import APIRouter, Path, HTTPException, status
from models import Todo, TodoItemResponse, TodoItemsResponse

todo_router = APIRouter()
todo_list = []

# Override default status code for resource creation
@todo_router.post("/todo", status_code=status.HTTP_201_CREATED)
async def add_todo(todo: Todo) -> dict:
    todo_list.append(todo)
    return {"message": "Todo added successfully."}

# Apply the response_model to filter the output
@todo_router.get("/todo", response_model=TodoItemsResponse)
async def retrieve_todos() -> dict:
    return {"todos": todo_list}

# Handle errors gracefully using HTTPException
@todo_router.get("/todo/{todo_id}", response_model=TodoItemResponse)
async def get_single_todo(todo_id: int = Path(..., title="The ID of the todo to retrieve")) -> dict:
    for todo in todo_list:
        if todo.id == todo_id:
            return todo

    # Raise an exception if the item is not found
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail="Todo with supplied ID doesn't exist."
    )
```
