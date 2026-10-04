# 15. Configuration, logging, and debugging

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Testing FastAPI applications](./14-testing-fastapi-applications.md) | [Notes index](../README.md) | [Next: Deployment, performance, and operations](./16-deployment-performance-and-operations.md) |

## Keep environment-specific settings outside source code

Development, test, and production environments need different database URLs, origins, and debug settings. Read these values from environment configuration instead of hard-coding credentials or editing source files between deployments.

Install Pydantic Settings:

~~~sh
python -m pip install pydantic-settings python-dotenv
~~~

Define validated settings in one place:

~~~python
from functools import lru_cache

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    app_name: str = "Books API"
    debug: bool = False
    database_url: str
    allowed_origins: list[str] = Field(default_factory=list)

    model_config = SettingsConfigDict(
        env_prefix="BOOKS_",
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )


@lru_cache
def get_settings() -> Settings:
    return Settings()
~~~

Environment values are strings at the operating-system boundary. Pydantic Settings parses and validates them against each declared type. With `env_prefix="BOOKS_"`, the `database_url` field reads `BOOKS_DATABASE_URL`.

The default factory creates an independent list for each settings instance.

## Configure local values without committing secrets

A local `.env` file can hold development-only values:

~~~dotenv
BOOKS_APP_NAME=Local Books API
BOOKS_DEBUG=true
BOOKS_DATABASE_URL=sqlite:///./notes.db
BOOKS_ALLOWED_ORIGINS=["http://localhost:5173"]
~~~

Add `.env` to `.gitignore`. Commit an `.env.example` with placeholder values and no working credentials so another developer knows which settings are required.

Production secrets should come from the deployment platform secret manager or protected environment variables. Do not store API keys, signing secrets, or database passwords in Git.

## Inject settings into the application

Use a dependency so tests and separate app instances can provide different settings:

~~~python
from typing import Annotated

from fastapi import Depends, FastAPI

app = FastAPI()
SettingsDependency = Annotated[Settings, Depends(get_settings)]


@app.get("/info")
async def app_info(settings: SettingsDependency):
    return {
        "name": settings.app_name,
        "debug": settings.debug,
    }
~~~

Do not return credentials or complete configuration objects from a public route. Expose only harmless values that clients need to know.

Tests can replace `get_settings` through `app.dependency_overrides`, as shown in the testing chapter. Clear the cache if a test directly changes environment variables after settings have already been loaded.

## Use the Python logging module

Create a named logger in each module. Configure logging once at the application entry point or through the server configuration:

~~~python
import logging

logger = logging.getLogger(__name__)


def load_catalog():
    logger.info("Loading catalog")
    try:
        return read_catalog_file()
    except OSError:
        logger.exception("Could not load catalog")
        raise
~~~

Use `DEBUG` for detailed diagnostics, `INFO` for normal events, `WARNING` for unexpected conditions, and `ERROR` for failed operations. `logger.exception()` includes the active traceback and should be called from an exception handler.

Avoid `print()` for application operation logs because it has no levels, structured context, or convenient filtering.

## Avoid leaking private data into logs

Do not log passwords, access tokens, session cookies, authorization headers, database connection strings, full payment details, or complete request bodies. Logs often have broader access and longer retention than the original request.

Log a request identifier, route name, safe resource identifier, outcome, and duration where useful. Sanitize user-controlled text so line breaks or control characters cannot forge misleading log entries.

## Read the server logs and inspect a request

Run Uvicorn with reload while developing:

~~~sh
python -m uvicorn app.main:app --reload --log-level debug
~~~

Reload watches files and restarts the development server. Do not use reload mode for production.

Use the interactive schema and a command-line client to separate route, validation, and browser issues:

~~~sh
curl.exe -i "http://127.0.0.1:8000/books?page=1"
curl.exe -v http://127.0.0.1:8000/health
~~~

Check the HTTP method, status, request headers, content type, response body, and server log entry. `/docs` displays the interactive schema when it is enabled, and `/openapi.json` returns the generated API schema.

## Use debug mode only in development

Debug mode can return diagnostic details that reveal implementation information. Keep it disabled in production and return safe error messages to clients. Send tracebacks to protected server logs instead.

Development and production can share the same application code while using different validated settings.

## Separate application settings from server settings

Application settings belong to the code, such as allowed origins or a database URL. Server settings belong to the ASGI server, such as bind address, port, worker count, and access logs.

Keep these groups distinct. Uvicorn CLI options configure the server process; Pydantic Settings configures values used by the application.

## Practice

1. Define required settings for the database URL and allowed origins.
2. Use a local `.env` file and confirm it is ignored by Git.
3. Inject settings through a dependency and override it in a test.
4. Add one informational log and one exception log with safe context.
5. Start the app with Uvicorn debug logs and trace one invalid request from client to server log.

## Main references

- [FastAPI settings documentation](https://fastapi.tiangolo.com/advanced/settings/)
- [Pydantic Settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
- [Python logging](https://docs.python.org/3/library/logging.html)
- [Uvicorn settings](https://www.uvicorn.org/settings/)
