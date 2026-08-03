# Python — FastAPI

FastAPI-specific skills: routing/validation, dependency injection, security/authentication, Redis caching, response/error handling, testing, and OpenAPI docs.

For REST API projects built on FastAPI. Pair with `python-core` and a database package.

## Install

```bash
npx skills add mihailtd/skills/skills/python-fastapi --all
```

Add `-g` to install globally instead of per-project. Use `--skill <name>` (repeatable) instead of `--all` to cherry-pick individual skills from this package, e.g.:

```bash
npx skills add mihailtd/skills/skills/python-fastapi --skill python-fastapi-api-testing
```

## Skills (10)

- **[python-fastapi-api-testing](python-fastapi-api-testing/SKILL.md)** — Instructs the agent on writing asynchronous unit tests for FastAPI REST API endpoints using pytest and httpx.AsyncClient, creating reusable test fixtures, and generating test coverage reports.
- **[python-fastapi-dependency-injection](python-fastapi-dependency-injection/SKILL.md)** — Instructs the agent on leveraging FastAPI's Depends class to create reusable dependency functions and inject them into path operation function arguments to enforce runtime conditions like database session preparation and authentication.
- **[python-fastapi-openapi-documentation](python-fastapi-openapi-documentation/SKILL.md)** — Ensures the agent generates robust, user-friendly automatic OpenAPI documentation (Swagger and ReDoc) by embedding example schemas, adding metadata to path parameters, and visually organizing routes with tags.
- **[python-fastapi-project-structuring](python-fastapi-project-structuring/SKILL.md)** — Guides the agent on organizing a growing FastAPI application into a clean, modular architecture by separating models, routes, and database configurations into standard directories and keeping the main.py entry point lightweight.
- **[python-fastapi-redis-caching](python-fastapi-redis-caching/SKILL.md)** — Instructs the agent on implementing the Cache-Aside pattern in FastAPI using Redis, both manually for complex logic and declaratively using the fastapi-cache library.
- **[python-fastapi-response-error-handling](python-fastapi-response-error-handling/SKILL.md)** — Instructs the agent on building customized response models using Pydantic, defining specific HTTP status codes, and handling errors gracefully using FastAPI's HTTPException.
- **[python-fastapi-routing-validation](python-fastapi-routing-validation/SKILL.md)** — Guides the agent on handling multiple routes modularly using the APIRouter class, validating request bodies using Pydantic models, and properly extracting path and query parameters in FastAPI.
- **[python-fastapi-partial-updates](python-fastapi-partial-updates/SKILL.md)** — Implementing PATCH-style partial updates using Pydantic's `exclude_unset` (backed by `model_fields_set`) to distinguish a field the client omitted from one explicitly set to null, since a naive full-object overwrite silently clears any field the calling client doesn't know about.
- **[python-fastapi-security-attack-resistance](python-fastapi-security-attack-resistance/SKILL.md)** — Instructs the agent on mitigating attacks through secure OS interactions, safe data parsing, preventing SQL injection, and configuring robust HTTP headers for XSS, CSRF, CORS, and Clickjacking defense in FastAPI.
- **[python-fastapi-security-authentication](python-fastapi-security-authentication/SKILL.md)** — Instructs the agent on all aspects of FastAPI security: protecting API routes using dependency injection, implementing OAuth2 with JWT for bearer token authentication, hashing user passwords with bcrypt or Argon2, secure cookie and session management, OAuth 2.0 Authorization Code flow with state parameter CSRF protection, authorization and access control, and configuring CORS middleware.
