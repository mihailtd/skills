---
name: fastapi-openapi-documentation
description: Ensures the agent generates robust, user-friendly automatic OpenAPI documentation (Swagger and ReDoc) by embedding example schemas, adding metadata to path parameters, and visually organizing routes with tags.
---

# FastAPI OpenAPI Documentation and Schema Guidelines

You are an expert Python developer specializing in FastAPI. When asked to build or document API routes, parameters, or data models, you must ensure the automatic documentation (Swagger and ReDoc) is robust and user-friendly by adhering to the following rules:

## 1. Pydantic Example Schemas

Always guide API users by providing concrete JSON examples of how they should format their request bodies.

- Use `model_config = ConfigDict(json_schema_extra={...})` inside your Pydantic `BaseModel` definitions to embed example payloads [2-4].
- This ensures FastAPI correctly generates the JSON Schema payload examples in the interactive documentation [5].

## 2. Path Parameter Metadata

Do not leave URL path parameters undocumented.

- Import and use the `Path` class from the `fastapi` library to distinguish path parameters and add descriptive metadata [6].
- Use the `title` argument within the `Path` function to provide a clear description of what the parameter does (e.g., `todo_id: int = Path(..., title="The ID of the todo to retrieve")`) [7].

## 3. Organizing Routes with Tags

Ensure the interactive documentation is visually organized by grouping related path operations together.

- When instantiating an `APIRouter`, always assign a descriptive tag using the `tags` argument (e.g., `user_router = APIRouter(tags=["User"])`) [8].
- This will group all routes defined under this router together on the Swagger UI page, making the API significantly easier to navigate.

---

## Code Examples

Below is a best-practice example demonstrating these documentation rules in action.

**models.py (Embedding Schema Examples)**

```python
from pydantic import BaseModel, ConfigDict

class Todo(BaseModel):
    model_config = ConfigDict(
        json_schema_extra={
            "examples": [
                {
                    "id": 1,
                    "item": "Study the OpenAPI documentation chapter",
                }
            ]
        }
    )

    id: int
    item: str
```

**routes.py (Adding Tags and Path Metadata)**

```python
from fastapi import APIRouter, Path
from models import Todo

# 3. Organize the interactive documentation visually using tags
todo_router = APIRouter(
    tags=["Todos"]
)

todo_list = []

@todo_router.post("/todo")
async def add_todo(todo: Todo) -> dict:
    todo_list.append(todo)
    return {"message": "Todo added successfully."}

# 2. Add titles and descriptive metadata directly to URL parameters
@todo_router.get("/todo/{todo_id}")
async def get_single_todo(
    todo_id: int = Path(..., title="The uniquely generated ID of the todo to retrieve")
) -> dict:
    for todo in todo_list:
        if todo.id == todo_id:
            return {"todo": todo}
    return {"message": "Todo doesn't exist."}
```
