---
name: python-restful-api-design
description: Instructs the agent on building standardized HTTP interfaces using resources and actions abstractions, proper HTTP methods/status codes, pagination, and the OpenAPI specification.
---

# Python RESTful API Design Guidelines

You are an expert Python developer specializing in software architecture and RESTful web services. When asked to design or implement HTTP interfaces, you must adhere to the following rules:

## 1. Resources and Actions Abstractions
Design the API around resources (passive elements identified by URIs) and actions (HTTP methods performed on them) using the CRUD (Create Retrieve Update Delete) approach.
* Use JSON format for most request and response bodies.
* Make the interface walkable by returning full URIs (hyperlinks) to related resources rather than indirect, no-context IDs.
* Maintain the same representation format for input and output, making it easy for a client to retrieve a resource, modify it, and resubmit it.

## 2. Proper Use of HTTP Methods and Status Codes
Map actions distinctly depending on whether the URI describes a collection or a single resource.
* **GET**: Use for retrieving a collection or single resource. Use query parameters exclusively for GET searches and filters.
* **POST**: Use to create a new element in a collection. This is generally the only non-idempotent action.
* **PUT / PATCH**: Use PUT to update/overwrite an entire resource, and PATCH for partial updates.
* **Status Codes**: Provide precise status codes rather than falling back to generic ones. Use `201 Created` for successful POST requests, `204 No Content` for successful DELETE requests, and `404 Not Found` for missing URIs. When returning a `400 Bad Request` or similar `4XX` error, always include a descriptive message in the body explaining why the request failed.

## 3. Pagination
Any `LIST` request that returns a sensible number of elements must be paginated to avoid slow response times and wasted bandwidth.
* Use query parameters like `page` and `size` to limit the scope of the request.
* Return a structured JSON response containing `next` and `previous` fields (providing hyperlinks to adjacent pages or `null`), and a `result` field containing the array of elements.

## 4. Open API Specification
Design and define the API explicitly using the Open API specification.
* Create definitions using YAML or JSON documents.
* Group reusable data structures under the `components: schemas:` section, and reference them in your endpoint definitions using `$ref`.

---

## Code Examples

Below are best-practice examples demonstrating how to structure resources, handle pagination, and define OpenAPI YAMLs.

**1. Standardized Pagination JSON Response**
```json
{
    "next": "http://api.example.com/search?page=2&size=10",
    "previous": null,
    "result": [
        {
            "id": 1,
            "href": "http://api.example.com/books/1",
            "name": "Frankenstein",
            "author": "Mary Shelley"
        }
    ]
}
2. Open API Specification (YAML)
openapi: 3.0.0
info:
  version: "1.0.0"
  title: "Example Books API"
paths:
  /books:
    post:
      summary: "Create a new book"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/Book"
      responses:
        "201":
          description: "Created"
        "400":
          description: "Invalid input"
components:
  schemas:
    Book:
      type: "object"
      properties:
        name:
          type: "string"
        author:
          type: "string"
```