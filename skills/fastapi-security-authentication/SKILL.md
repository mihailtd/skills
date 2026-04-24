---
name: fastapi-security-authentication
description: Instructs the agent on all aspects of FastAPI security: protecting API routes using dependency injection, implementing OAuth2 with JWT for bearer token authentication, hashing user passwords with bcrypt or Argon2, secure cookie and session management, OAuth 2.0 Authorization Code flow with state parameter CSRF protection, authorization and access control, and configuring CORS middleware.
---

# FastAPI Security and Authentication Guidelines

You are an expert Python developer specializing in FastAPI security. When asked to secure a FastAPI application, implement authentication, authorization, session management, or configure middleware, you must adhere to the following rules:

---

## 1. Password Hashing

Never store passwords in plain text or encrypt them with standard symmetric ciphers.

- Always use a slow, resource-intensive Key Derivation Function (KDF) to resist brute-force and GPU cracking attacks.
- **Preferred:** Use **Argon2** (via `passlib` with `argon2-cffi`) as it is both memory-intensive and computationally intensive.
- **Acceptable:** Use `bcrypt` via `passlib[bcrypt]` when Argon2 is not available.
- Create a `CryptContext` instance (e.g., `CryptContext(schemes=["bcrypt"], deprecated="auto")`) to handle hashing and verification.
- Ensure every password hash uses a unique, random salt (handled automatically by passlib) to defeat rainbow table attacks.
- Use the context to hash passwords during user sign-up and verify them against the hashed strings during sign-in.

---

## 2. OAuth2 and JSON Web Tokens (JWT)

Use the OAuth2 password flow with bearer token authentication for managing user sessions within the app.

- Provide an authentication endpoint utilizing `OAuth2PasswordRequestForm` from `fastapi.security` to extract the client's username and password.
- Generate a JWT when credentials are valid. Use the `jose` library (`jwt.encode`) to sign the token with a secret key and the `HS256` algorithm.
- Include the user's identifier and an expiration time (`expires`) in the JWT payload.
- Ensure access tokens have a short, strict expiration time (e.g., 3600 seconds).

---

## 3. OAuth 2.0 Authorization Code Flow (Third-Party Providers)

When integrating with third-party OAuth 2.0 providers (e.g., Google, GitHub) or building an authorization server, strictly follow these rules:

- **Grant Type:** Only use the **Authorization Code flow** for websites, mobile apps, and browser-based apps. Never use the deprecated Implicit flow.
- **State Parameter:** Always generate and validate a random `state` parameter during the authorization flow. The client must verify that the returning `state` matches the one sent, to prevent CSRF and session fixation attacks.
- **HTTPS Only:** Enforce HTTPS for all OAuth redirect URIs to prevent authorization codes from being intercepted.
- **Token Scope:** Request and grant only the scopes absolutely necessary (Principle of Least Privilege).

---

## 4. Protecting Routes via Dependency Injection

Do not manually check tokens inside every route body. Use FastAPI's Dependency Injection system to authorize requests automatically.

- Create a reusable dependency using `OAuth2PasswordBearer` that extracts the token from the `Authorization: Bearer` header.
- Write an authentication function that decodes the token (`jwt.decode`) and verifies its validity and expiration time.
- Inject this authentication dependency into protected routes using `Depends()` (e.g., `user: str = Depends(authenticate)`).
- Raise an `HTTPException` with `status.HTTP_401_UNAUTHORIZED` or `status.HTTP_403_FORBIDDEN` if token validation fails.

---

## 5. Secure Cookie & Session Management

When issuing cookies for session IDs or authentication tokens, protect them from unauthorized access and cross-site attacks.

- **HttpOnly Directive:** Always set `httponly=True` to hide the cookie from client-side JavaScript, mitigating session hijacking via XSS.
- **Secure Directive:** Always set `secure=True` so the browser only transmits the cookie over HTTPS, preventing interception by network eavesdroppers.
- **SameSite Directive:** Always set `samesite="lax"` or `samesite="strict"` to prevent CSRF attacks. Never use `samesite="none"` for sensitive cookies.
- **Serialization:** Never use `pickle` to serialize session state — this is vulnerable to remote code execution. Always use safe formats like JSON.

---

## 6. Authorization & Access Control

Authentication verifies identity; authorization verifies permissions. These are distinct concerns and both must be enforced.

- **Principle of Least Privilege (PLP):** Ensure users and systems are given only the minimal permissions needed to perform their responsibilities.
- **Server-Side Enforcement:** Never rely on UI manipulation (e.g., hiding or disabling buttons) as a security measure. Always enforce authorization checks server-side before processing any unsafe request.

---

## 7. Cross-Origin Resource Sharing (CORS) Middleware

Ensure the API can be securely consumed by frontend applications by handling cross-origin HTTP requests.

- Import `CORSMiddleware` from `fastapi.middleware.cors`.
- Register the middleware using `app.add_middleware()`.
- Explicitly configure `allow_origins` (a list of permitted URLs, or `["*"]` for all), `allow_credentials`, `allow_methods`, and `allow_headers`.

---

## Code Examples

### `hash_password.py` — Password Hashing (bcrypt)

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class HashPassword:
    def create_hash(self, password: str):
        return pwd_context.hash(password)

    def verify_hash(self, plain_password: str, hashed_password: str):
        return pwd_context.verify(plain_password, hashed_password)
```

---

### `jwt_handler.py` — JWT Token Management

```python
import time
from jose import jwt, JWTError
from fastapi import HTTPException, status

SECRET_KEY = "YOUR_SUPER_SECRET_KEY"

def create_access_token(user: str) -> str:
    payload = {
        "user": user,
        "expires": time.time() + 3600
    }
    return jwt.encode(payload, SECRET_KEY, algorithm="HS256")

def verify_access_token(token: str) -> dict:
    try:
        data = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        expire = data.get("expires")
        if expire is None or time.time() > expire:
            raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Token expired or missing")
        return data
    except JWTError:
        raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="Invalid token")
```

---

### `authenticate.py` — Dependency Injection

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jwt_handler import verify_access_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/user/signin")

async def authenticate(token: str = Depends(oauth2_scheme)) -> str:
    if not token:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Sign in for access")
    decoded_token = verify_access_token(token)
    return decoded_token["user"]
```

---

### `main.py` — Wiring Routes, CORS, and Secure Cookies

```python
from fastapi import FastAPI, Depends, Response
from fastapi.middleware.cors import CORSMiddleware
from authenticate import authenticate

app = FastAPI()

# Configure CORS Middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Protect routes by injecting the 'authenticate' dependency
@app.post("/event/new")
async def create_event(event_data: dict, user: str = Depends(authenticate)) -> dict:
    return {"message": f"Event created successfully by {user}"}

# Secure cookie example
@app.post("/login")
async def login(response: Response):
    token = "generated_secure_random_token"
    response.set_cookie(
        key="session_id",
        value=token,
        httponly=True,
        secure=True,
        samesite="lax",
        max_age=3600,
    )
    return {"message": "Successfully logged in"}
```

---

### `oauth_router.py` — OAuth 2.0 Authorization Code Flow (using `authlib`)

Use the `authlib` library with its async httpx integration for OAuth 2.0 flows.
This handles state generation, PKCE, token refresh, and protocol details correctly.

Install: `uv add authlib httpx`

```python
from authlib.integrations.httpx_client import AsyncOAuth2Client
from fastapi import APIRouter, Request, HTTPException, status
from fastapi.responses import RedirectResponse

oauth_router = APIRouter()

CLIENT_ID = "your_client_id"
CLIENT_SECRET = "your_client_secret"
AUTH_URL = "https://provider.com/o/authorize"
TOKEN_URL = "https://provider.com/o/token"
REDIRECT_URI = "https://your-app.com/callback"

def _create_oauth_client() -> AsyncOAuth2Client:
    return AsyncOAuth2Client(
        client_id=CLIENT_ID,
        client_secret=CLIENT_SECRET,
        redirect_uri=REDIRECT_URI,
    )

@oauth_router.get("/login/oauth")
async def login_oauth(request: Request):
    client = _create_oauth_client()

    # authlib generates the state parameter automatically for CSRF protection
    authorization_url, state = client.create_authorization_url(AUTH_URL)

    # Save state securely (e.g., in a secure session cookie or Redis) to verify later
    request.session["oauth_state"] = state
    return RedirectResponse(authorization_url)

@oauth_router.get("/callback")
async def oauth_callback(request: Request, code: str, state: str):
    # Validate the state parameter to prevent CSRF and session fixation attacks
    saved_state = request.session.get("oauth_state")
    if not saved_state or state != saved_state:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Invalid state parameter",
        )

    # Securely exchange the authorization code for an access token (async)
    client = _create_oauth_client()
    token = await client.fetch_token(
        TOKEN_URL,
        code=code,
        state=state,
    )
    await client.aclose()

    return {"access_token": token["access_token"]}
```
