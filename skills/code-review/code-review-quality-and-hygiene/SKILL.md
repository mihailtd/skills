---
name: code-review-quality-and-hygiene
description: Review URL construction (urllib, not string concatenation), descriptive variable naming, single-BASE_URL-per-integration hygiene, and environment-variable-vs-hardcoded-value decisions. Use when implementing or reviewing API clients, integrations, or agent/automation code that calls HTTP endpoints.
compatibility: "Python 3.x (stdlib only: urllib.parse, os)"
metadata:
  category: python-style
  focus: code-quality
---

## Purpose

Prevent broken URLs, reduce environment-switching mistakes, and improve readability/maintainability of integration code.

## When to use this skill

Use this skill when:
- writing or reviewing Python code that calls HTTP APIs
- defining integration configuration (base URLs, endpoints, cron schedules)
- naming variables in integration/client code

## Requirements (normative)

### 1) Construct URLs with `urllib.parse`, not string concatenation

**MUST** use `urllib.parse.urljoin` to append paths to a base URL.
**MUST NOT** build URLs via string concatenation (e.g., `base_url + "/path"`).

**Why:** string concatenation commonly creates missing slashes, double slashes, and broken URL escaping.

**Example (path joining):**
```py
from urllib.parse import urljoin

BASE_URL = "https://example.com/"   # trailing slash recommended
endpoint = "users/profile"          # no leading slash recommended

full_url = urljoin(BASE_URL, endpoint)
print(full_url)  # "https://example.com/users/profile"
```

**MUST** use `urllib.parse.urlencode` for query parameters (do not manually assemble `?a=1&b=2`).

**Example (query encoding):**
```py
from urllib.parse import urljoin, urlencode

BASE_URL = "https://example.com/"
endpoint = "users/profile"

params = {"active": "true", "limit": 50}

full_url = urljoin(BASE_URL, endpoint) + "?" + urlencode(params)
print(full_url)  # "https://example.com/users/profile?active=true&limit=50"
```

**Common pitfall:** `urljoin` treats a leading `/` as an absolute path and may replace a base path.
**Standardize on:**
- `BASE_URL` ends with `/`
- endpoints do **not** start with `/`

---

### 2) Use full variable names (no acronyms/abbreviations)

**MUST** use descriptive names that communicate intent and domain meaning.

bad example: `iam`
good example: `identityProviderClient`

bad example: `chkRes`
good example: `checkResult`

**Guideline:** prefer clear names even if longer. Optimize for readers and maintainers.

For the deeper, research-backed treatment of naming (why it matters, single-letter variable risk, camelCase vs. snake_case, and codebase-wide naming consistency), see code-review-naming-fundamentals, code-review-naming-word-choice, and code-review-naming-consistency.

---

### 3) `BASE_URL` must be only the base (no path, no query)

**MUST** define `BASE_URL` as scheme + host (and optional port) only.
**MUST NOT** include path segments (e.g., `/latest/`) or query parameters.

**MUST** keep **a single `BASE_URL` per integration**.

**Anti-pattern (do not do this):**
```json
"VENDOR_APPROVALS_URL": "https://tenant-dev.vendor-identity.example.com/ui/d/approvals",
"VENDOR_BASE_URL": "https://tenant.api.vendor-identity.example.com/latest/",
"VENDOR_TOKEN_URL": "https://tenant.api.vendor-identity.example.com/oauth/token",
```

**WHY?** When switching environments, you would need to update multiple values. This increases the chance of partial updates and production misconfiguration.

**Correct pattern:**
```py
from urllib.parse import urljoin

BASE_URL = "https://tenant.api.vendor-identity.example.com/"

approvals_endpoint = "ui/d/approvals"
latest_endpoint = "latest/"
token_endpoint = "oauth/token"

full_approvals_url = urljoin(BASE_URL, approvals_endpoint)
full_latest_url = urljoin(BASE_URL, latest_endpoint)
full_token_url = urljoin(BASE_URL, token_endpoint)

print(full_latest_url)
print(full_approvals_url)
print(full_token_url)
```

---

### 4) Decide: environment variable vs hardcoded value

A configuration value **SHOULD** be an environment variable if **any** of the following are true:

1. **Different values per environment**
   - different cron schedules in dev vs prod
   - different base URLs per environment (e.g., a sandbox tenant in dev, the production tenant in prod)

2. **Need to change quickly without redeploy**
   - change a cron schedule via environment configuration (e.g., a secrets manager or config service) instead of code + redeploy
   - switch a base URL quickly via environment configuration

3. **Secret or sensitive**
   - passwords, tokens, API keys
   - sensitive identifiers (including email addresses) if policy prohibits hardcoding them

If the answer to **all** is no, the value **MAY** be hardcoded.

**Example (BASE_URL from environment):**

```py
import os
from urllib.parse import urljoin

BASE_URL = os.environ["VENDOR_BASE_URL"].rstrip("/") + "/"

approvals_endpoint = "ui/d/approvals"
full_approvals_url = urljoin(BASE_URL, approvals_endpoint)

print(full_approvals_url)
```

## Review checklist (quick)

- [ ] No URL string concatenation for paths
- [ ] `urljoin` used for path composition
- [ ] `urlencode` used for query parameters
- [ ] One `BASE_URL` per integration; contains no paths/queries
- [ ] Endpoints are relative (no leading `/` by convention)
- [ ] Variable names are descriptive (no cryptic abbreviations)
- [ ] Env vars used when values vary by environment, must be changed quickly, or are sensitive

## Examples

### Good: consistent URL building

```py
from urllib.parse import urljoin, urlencode
import os

BASE_URL = os.environ["INTEGRATION_BASE_URL"].rstrip("/") + "/"

ENDPOINTS = {
    "profile": "users/profile",
    "approvals": "ui/d/approvals",
    "latest": "latest/",
}

def build_url(endpoint_key: str, query: dict | None = None) -> str:
    url = urljoin(BASE_URL, ENDPOINTS[endpoint_key])
    if query:
        url += "?" + urlencode(query)
    return url

print(build_url("profile", {"active": "true"}))
```

### Bad: brittle concatenation (avoid)

```py
BASE_URL = "https://example.com"
full_url = BASE_URL + "/users/profile"  # breaks easily with slashes
```

### Naming examples

bad: `iam`, `chkRes`, `cfg`, `tmp`
good: `identityProviderClient`, `checkResult`, `integrationConfig`, `temporaryToken`
