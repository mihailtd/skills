---
name: python-pydantic-configuration
description: Instructs the agent on handling application configuration using Pydantic BaseSettings, validating environment variables, managing .env files, and adhering to the Twelve-Factor App methodology.
---

# Python Pydantic Configuration & Environment Guidelines

You are an expert Python developer specializing in secure, scalable application configuration. When asked to implement configuration management, environment variables, or application settings, you must combine Pydantic's validation with Twelve-Factor App principles and adhere to the following rules:

## 1. The Twelve-Factor App Methodology
Strictly separate configuration from the codebase [1, 2]. 
*   All operational configuration (e.g., database URLs, credentials, logging levels) and sensitive data must be stored in environment variables, never hardcoded [3-5]. 
*   Treat configuration as independent from the code so the same codebase can be deployed across different environments (development, staging, production) simply by changing the environment variables [1, 6].

## 2. Use Pydantic `BaseSettings`
Bridge the gap between raw OS environment variables (`os.environ`) and your application logic by using Pydantic for run-time validation and model consistency [7, 8].
*   Subclass Pydantic's `BaseSettings` to define your configuration schema [9, 10].
*   Use type hints (e.g., `str`, `int`, `Optional[str]`) for every configuration variable to enforce strict data validation [10]. 

## 3. Local `.env` File Handling
For local development, manage environment variables using a `.env` file [9, 10].
*   Configure your `BaseSettings` subclass to automatically read from this file by declaring an inner `Config` class with the attribute `env_file = ".env"` [9, 10]. 
*   Ensure that the `.env` file is excluded from version control (e.g., via `.gitignore`) to protect sensitive data [3, 11].

## 4. Strict Config Validation and Security
Leverage Pydantic's advanced validation features to ensure your application fails quickly and loudly if the configuration is invalid or missing [12, 13].
*   Use specific Pydantic types like `EmailStr` or `AnyHttpUrl` and utilize `Field` constraints to sanitize incoming configuration values and prevent injection risks [14-16].

---

## Code Examples

Below are best-practice examples demonstrating how to implement secure, validated configuration.

**1. Defining the Configuration Model (`config.py`)**
```python
from typing import Optional
from pydantic import BaseSettings, Field, EmailStr

# BaseSettings automatically reads from environment variables and parses them
class Settings(BaseSettings):
    # Fails loudly if missing or not a valid URL
    DATABASE_URL: str = Field(..., description="PostgreSQL connection string")
    
    # Uses specific types for strict validation
    ADMIN_EMAIL: EmailStr
    
    # Optional feature configurations
    DEBUG_MODE: bool = False
    SECRET_KEY: Optional[str] = None

    # Instructs Pydantic to load variables from a local .env file
    class Config:
        env_file = ".env"

# Instantiate the settings to be used throughout the app
settings = Settings()
2. Generating a Local .env File (Terminal)
# Safely create a local .env file for development (do not commit this file)
$ touch .env
$ echo "DATABASE_URL=postgresql://user:pass@localhost:5432/mydb" >> .env
$ echo "ADMIN_EMAIL=admin@example.com" >> .env
$ echo "DEBUG_MODE=True" >> .env
3. Using the Configuration Securely (main.py)
from fastapi import FastAPI
from .config import settings
import logging

app = FastAPI()

# The application safely uses validated, strongly-typed variables
@app.on_event("startup")
async def startup_event():
    if settings.DEBUG_MODE:
        logging.getLogger().setLevel(logging.DEBUG)
        logging.debug("Running in debug mode.")
        
    logging.info(f"Admin contact: {settings.ADMIN_EMAIL}")
    # Initialize database using settings.DATABASE_URL securely here...
```