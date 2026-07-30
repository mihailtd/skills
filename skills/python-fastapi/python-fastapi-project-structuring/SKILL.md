---
name: python-fastapi-project-structuring
description: Guides the agent on organizing a growing FastAPI application into a clean, modular architecture by separating models, routes, and database configurations into standard directories and keeping the main.py entry point lightweight.
---

# FastAPI Project Structuring Guidelines

You are an expert Python developer specializing in FastAPI application architecture. When asked to build, structure, or refactor a growing FastAPI application, you must adhere to the following rules:

## 1. Modular Architecture

Avoid writing all application code in a single file. Arrange application components in an organized, modular format to improve readability, development speed, and scalability.

- Create standard application subdirectories:
  - `models/` for data and response models.
  - `routes/` for API endpoint path operations.
  - `database/` for database connection instances and configurations.

## 2. Python Modules

- Always create an `__init__.py` file inside each subdirectory (e.g., `models/__init__.py`, `routes/__init__.py`, `database/__init__.py`). This is required to treat the folders as Python modules and allow absolute imports across the project.

## 3. Lightweight `main.py` Entry Point

Keep the `main.py` entry point file as lightweight as possible.

- Do not define your route path operations directly in `main.py`.
- Define all routes using the `APIRouter` class within files in the `routes/` module.
- In `main.py`, initialize the core `FastAPI()` instance and simply make the modular routes visible by registering them using `app.include_router()`.

---

## Example Architecture

Below is a standard directory structure and a best-practice `main.py` implementation demonstrating these concepts.

**Project Directory Structure:**

```text
planner/
├── main.py
├── database/
│   ├── __init__.py
│   └── connection.py
├── routes/
│   ├── __init__.py
│   ├── events.py
│   └── users.py
└── models/
    ├── __init__.py
    ├── events.py
    └── users.py
main.py (Wiring modular routes together):
from fastapi import FastAPI
from routes.users import user_router
from routes.events import event_router
import uvicorn

app = FastAPI()

# Register routes imported from the modular routes/ directory
app.include_router(user_router, prefix="/user")
app.include_router(event_router, prefix="/event")

if __name__ == '__main__':
    uvicorn.run("main:app", host="0.0.0.0", port=8080, reload=True)
```
