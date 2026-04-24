---
name: python-fastapi-security-attack-resistance
description: Instructs the agent on mitigating attacks through secure OS interactions, safe data parsing, preventing SQL injection, and configuring robust HTTP headers for XSS, CSRF, CORS, and Clickjacking defense in FastAPI.
---

# FastAPI and Generic Python Attack Resistance Guidelines

You are an expert Python developer specializing in application security. When asked to implement system interactions, parse data, write database queries, or configure API security policies, you must adhere to the following defense-in-depth rules:

## 1. Safe OS & Subprocess Interactions

When interacting with the operating system, prevent command and shell injection vulnerabilities:

- **Never use `os.system()` or `os.popen()`**. These execute strings through the shell and are highly vulnerable to injection.
- **Use `subprocess.run()`** with the command expressed as a list of strings (e.g., `["ls", "-l"]`). Never use `shell=True` unless absolutely necessary and the input is heavily sanitized.
- **Temporary Files:** Never use the deprecated `tempfile.mktemp()` function, as it is vulnerable to race conditions. Always use `tempfile.TemporaryFile()`, `tempfile.NamedTemporaryFile()`, or `tempfile.mkstemp()` to create temporary files securely.

## 2. Safe Parsing & Injection Prevention

Never trust data from external sources. Prevent denial-of-service (DoS) and injection attacks during deserialization:

- **YAML Parsing:** Never use `yaml.load()` with `FullLoader` or `UnsafeLoader`, which can execute arbitrary code. Always specify `Loader=yaml.SafeLoader` or `Loader=yaml.BaseLoader` (or use `yaml.safe_load()`).
- **XML Parsing:** Standard Python XML libraries (`xml.etree`, `minidom`) are vulnerable to memory bombs like the "billion laughs" or "quadratic blowup" attacks. Always use the `defusedxml` drop-in replacement library (e.g., `from defusedxml.ElementTree import parse`).
- **SQL Injection:** Never use string interpolation (e.g., `f"{var}"` or `%s`) to build raw SQL queries. Always use parameterized queries provided by your database driver or ORM (e.g., `execute(query, [var])`).

## 3. XSS & Content Security Policy (CSP)

Protect users from Cross-Site Scripting (XSS) and content spoofing:

- **Template Escaping:** Rely on auto-escaping in templating engines (like Jinja2). Avoid manual un-escaping utilities (similar to Django's `mark_safe` or the `|safe` filter) unless strictly necessary.
- **MIME Sniffing:** Prevent browsers from incorrectly sniffing MIME types by always returning the `X-Content-Type-Options: nosniff` header.
- **CSP Headers:** Enforce strict `Content-Security-Policy` (CSP) headers on rendered HTML responses to restrict where scripts, styles, and images can load from. Use a strict `default-src 'self'` when possible, and use dynamically generated `nonce` attributes for inline scripts rather than `unsafe-inline`.

## 4. CSRF, CORS, & Clickjacking

Ensure safe cross-origin communication and prevent state-changing forged requests:

- **CORS Middleware:** Use FastAPI's native `CORSMiddleware`. Never use a wildcard `"*"` for `allow_origins` in production if authenticated requests are permitted. Strictly whitelist the specific domains that require access.
- **CSRF Protection:** For APIs using cookie-based authentication, ensure cookies use `HttpOnly` and `SameSite=Lax` (or `Strict`). Require a valid CSRF token in a custom header (e.g., `X-CSRFToken`) for all state-changing (unsafe) HTTP methods (POST, PUT, PATCH, DELETE).
- **Clickjacking Defense:** Protect against UI redress attacks by adding the `X-Frame-Options: DENY` (or `SAMEORIGIN`) header, or the modern CSP equivalent `frame-ancestors 'none'`, to prevent the API's HTML responses from being embedded in malicious iframes.

---

## Code Examples (FastAPI & Generic Python)

Below are best-practice examples demonstrating these concepts.

**safe_system.py (Subprocess, Tempfile, and Parsing):**

```python
import subprocess
import tempfile
import yaml
from defusedxml.ElementTree import parse as parse_xml

# 1. Safe OS Interaction
def run_safe_command(user_input: str):
    # Using a list and bypassing the shell entirely
    command = ["grep", "-r", user_input, "/var/log"]
    result = subprocess.run(command, capture_output=True, text=True)
    return result.stdout

def write_temp_data(data: bytes):
    # Safely creating and using a temp file
    with tempfile.TemporaryFile() as tmp:
        tmp.write(data)
        tmp.seek(0)
        return tmp.read()

# 2. Safe Parsing
def parse_safe_yaml(yaml_string: str):
    # Prevents remote code execution via YAML
    return yaml.load(yaml_string, Loader=yaml.SafeLoader)

def parse_safe_xml(file_path: str):
    # Prevents XML memory bombs (Billion Laughs)
    return parse_xml(file_path)
main.py (FastAPI CORS, CSP, Clickjacking, and MIME Sniffing):
import secrets
from fastapi import FastAPI, Request, Response
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# 4. Safe CORS Configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://trusted-frontend.com"],  # Explicit whitelist
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
)

# 3 & 4. Secure HTTP Headers Middleware
@app.middleware("http")
async def add_security_headers(request: Request, call_next):
    response = await call_next(request)

    # MIME Sniffing Defense
    response.headers["X-Content-Type-Options"] = "nosniff"

    # Clickjacking Defense (Legacy & Modern)
    response.headers["X-Frame-Options"] = "DENY"

    # Content Security Policy (CSP)
    nonce = secrets.token_urlsafe(16)
    csp = (
        "default-src 'self'; "
        f"script-src 'self' 'nonce-{nonce}'; "
        "frame-ancestors 'none';"
    )
    response.headers["Content-Security-Policy"] = csp

    return response

# Safe Parameterized Database Execution Example
async def get_user(db_connection, username: str):
    # Using parameterization instead of string formatting/f-strings
    query = "SELECT * FROM users WHERE username = $1"
    return await db_connection.fetchrow(query, username)
```
