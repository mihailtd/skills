---
name: python-functional-web-services
description: >
  Instructs the agent on designing FastAPI route handlers as stateless, functional
  pipelines. It guides decomposing handlers into a cascade of pure functions for
  validation, data retrieval, and response preparation using Pydantic models and
  HTTPException.
---

# Python Functional Web Services Guidelines

You are an expert Python developer specializing in functional programming and RESTful API design. When building web services with FastAPI, design route handlers as stateless functional pipelines. Adhere to the following rules:

## 1. Stateless REST APIs

Treat the HTTP request-response cycle as a pure function: `response = f(request)`.

- Avoid relying on server-side session state or stateful cookies for REST APIs.
- Each request should contain all necessary information (e.g., bearer tokens in headers) to authorize and process the transaction independently.

## 2. Decompose Route Handlers

Do not write monolithic route handlers that intermingle data access, business logic, and presentation. A FastAPI route handler must only do three things in a strict sequence:

1. **Validate:** Accept and validate request data via Pydantic models, path/query parameters, and dependencies.
2. **Get Data:** Delegate to a separate, pure data-service function to retrieve and filter the underlying data.
3. **Prepare Response:** Return a typed Pydantic response model. Let FastAPI handle serialization.

## 3. Forgiveness over Permission (EAFP) Error Handling

When filtering or fetching data, focus on the "happy path".

- Do not write excessively defensive `if` statements to check for missing data or bad keys.
- Instead, use `try...except` blocks to catch expected exceptions (like `KeyError` or `ValueError`) and immediately translate them into an HTTP error response using `raise HTTPException(status_code=404)`.

## 4. Functional Serialization with Pydantic

FastAPI's integration with Pydantic provides built-in functional serialization.

- Define `response_model` on route decorators so FastAPI automatically serializes your return value.
- Write pure transformation functions that convert domain objects to Pydantic response models.
- For multiple output formats (JSON, CSV), write pure serialization functions and select based on the `Accept` header or a query parameter.

---

## Code Examples

**1. Functional Serialization with Pydantic Response Models**

```python
from pydantic import BaseModel

class DataPoint(BaseModel):
    x: float
    y: float

class DatasetResponse(BaseModel):
    series_id: str
    points: list[DataPoint]

# Pure transformation function
def to_dataset_response(series_id: str, raw_data: list[dict]) -> DatasetResponse:
    points = [DataPoint(**p) for p in raw_data]
    return DatasetResponse(series_id=series_id, points=points)
```

**2. Three-Step Route Handler (FastAPI)**

```python
from fastapi import APIRouter, HTTPException

router = APIRouter()

# Pure data-service function (abstracted away from the web tier)
DB: dict[str, list[dict]] = {
    "I": [{"x": 10.0, "y": 8.04}],
    "II": [{"x": 8.0, "y": 6.95}],
}

def get_dataset(series_id: str) -> list[dict]:
    return DB[series_id]  # Will intentionally raise KeyError if invalid

@router.get("/dataset/{series_id}", response_model=DatasetResponse)
async def series_view(series_id: str) -> DatasetResponse:
    # 1. Validate — series_id comes pre-validated from the path parameter

    # 2. Get Data (with EAFP error handling)
    try:
        raw_data = get_dataset(series_id)
    except KeyError:
        raise HTTPException(status_code=404, detail="Unknown Series")

    # 3. Prepare Response (delegate to the pure transformation function)
    return to_dataset_response(series_id, raw_data)
```

**3. Multi-Format Serialization**

```python
import csv
import io
import json
from collections.abc import Callable
from typing import Any
from functools import wraps

from fastapi import APIRouter, Query, Response

router = APIRouter()

# Decorator to cleanly separate string generation from byte encoding
def to_bytes(function: Callable[..., str]) -> Callable[..., bytes]:
    @wraps(function)
    def decorated(*args: Any, **kwargs: Any) -> bytes:
        text = function(*args, **kwargs)
        return text.encode("utf-8")
    return decorated

@to_bytes
def serialize_json(data: list[dict[str, Any]]) -> str:
    return json.dumps(data, sort_keys=True)

@to_bytes
def serialize_csv(data: list[dict[str, Any]]) -> str:
    if not data:
        return ""
    output = io.StringIO()
    writer = csv.DictWriter(output, fieldnames=data[0].keys())
    writer.writeheader()
    writer.writerows(data)
    return output.getvalue()

# Registry mapping formats to their pure serializer functions
SERIALIZERS: dict[str, tuple[Callable, str]] = {
    "json": (serialize_json, "application/json"),
    "csv": (serialize_csv, "text/csv"),
}

@router.get("/export/{series_id}")
async def export_series(series_id: str, fmt: str = Query(default="json")) -> Response:
    try:
        raw_data = get_dataset(series_id)
    except KeyError:
        raise HTTPException(status_code=404, detail="Unknown Series")

    serializer, content_type = SERIALIZERS.get(fmt, SERIALIZERS["json"])
    content_bytes = serializer(raw_data)
    return Response(content=content_bytes, media_type=content_type)
```
