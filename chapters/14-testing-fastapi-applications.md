# 14. Testing FastAPI applications

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Background tasks and application lifespan](./13-background-tasks-and-lifespan.md) | [Notes index](../README.md) | [Next: Configuration, logging, and debugging](./15-configuration-logging-and-debugging.md) |

## Test the API through HTTP requests

FastAPI provides `TestClient`, which sends requests to an application in process. Tests can check status codes, response JSON, headers, and validation without starting a web server.

Install the test tools in the project environment:

~~~sh
python -m pip install pytest httpx
~~~

## Create an app that is easy to test

An application factory returns a fresh app and isolated data for each test:

~~~python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, Field


class BookCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)


class BookPublic(BaseModel):
    id: int
    title: str


def create_app():
    app = FastAPI()
    books = {}

    @app.post("/books", response_model=BookPublic, status_code=status.HTTP_201_CREATED)
    async def create_book(payload: BookCreate):
        book_id = len(books) + 1
        book = {"id": book_id, "title": payload.title}
        books[book_id] = book
        return book

    @app.get("/books/{book_id}", response_model=BookPublic)
    async def read_book(book_id: int):
        book = books.get(book_id)
        if book is None:
            raise HTTPException(status_code=404, detail="Book not found")
        return book

    return app
~~~

Keeping setup inside a function gives each test a new app and an empty store. A larger application can also inject repositories, settings, and service objects through dependencies.

## Test success and error responses

Create a test file such as `tests/test_books.py`:

~~~python
from fastapi.testclient import TestClient

from app import create_app


def test_create_and_read_book():
    client = TestClient(create_app())

    created = client.post("/books", json={"title": "The Hobbit"})
    assert created.status_code == 201
    assert created.json() == {"id": 1, "title": "The Hobbit"}

    response = client.get("/books/1")
    assert response.status_code == 200
    assert response.json()["title"] == "The Hobbit"


def test_missing_book_returns_404():
    client = TestClient(create_app())

    response = client.get("/books/404")

    assert response.status_code == 404
    assert response.json() == {"detail": "Book not found"}


def test_invalid_body_returns_422():
    client = TestClient(create_app())

    response = client.post("/books", json={"title": ""})

    assert response.status_code == 422
~~~

Run tests from the project root:

~~~sh
python -m pytest
~~~

Tests should check observable behavior such as status, response fields, and validation outcome. Avoid asserting an entire framework error sentence because library versions can change its wording.

## Use a client fixture

A pytest fixture can create a clean app and client for each test:

~~~python
import pytest
from fastapi.testclient import TestClient

from app import create_app


@pytest.fixture
def client():
    with TestClient(create_app()) as test_client:
        yield test_client


def test_empty_book_store(client):
    response = client.get("/books/1")
    assert response.status_code == 404
~~~

Using `TestClient` as a context manager runs application lifespan startup and shutdown. This matters when the app initializes a connection pool, cache, or another resource.

## Override a dependency in a test

Replace an external or protected dependency with a deterministic test value. Restore previous overrides so tests do not leak state:

~~~python
from fastapi.testclient import TestClient

from app import app, get_current_user


async def fake_current_user():
    return {"id": 7, "name": "Test Reader"}


def test_profile_uses_test_user():
    original_overrides = app.dependency_overrides.copy()
    app.dependency_overrides[get_current_user] = fake_current_user
    try:
        with TestClient(app) as client:
            response = client.get("/profile")
        assert response.status_code == 200
        assert response.json() == {"id": 7, "name": "Test Reader"}
    finally:
        app.dependency_overrides = original_overrides
~~~

Use a temporary database or repository override for tests that write data. Never point a destructive test at production data.

## Test asynchronous application code

Use an asynchronous HTTPX client when a test itself must await asynchronous operations:

~~~python
import pytest
from httpx import ASGITransport, AsyncClient

from app import app


@pytest.mark.anyio
async def test_health_endpoint():
    transport = ASGITransport(app=app)
    async with AsyncClient(
        transport=transport,
        base_url="http://test",
    ) as client:
        response = await client.get("/health")

    assert response.status_code == 200
~~~

`ASGITransport` sends requests to the app without opening a network port. Lifespan is not automatically triggered by every ASGI transport setup. If a test needs startup and shutdown, use a lifespan manager supported by the installed Starlette and HTTPX versions.

For most endpoint tests, synchronous `TestClient` is simpler and works with async path operations. Use the async client when the test needs to call other async functions or async resources directly.

## Separate test types

- Unit tests exercise a small function or service without HTTP.
- API tests use `TestClient` to verify routes and dependency behavior.
- Integration tests connect to a dedicated test database or external test service.
- Deployment checks verify environment configuration and the running server boundary.

Keep each test focused on one behavior. Give each test clean state and avoid depending on execution order.

## Check the generated OpenAPI schema

The application exposes its generated schema at `/openapi.json` by default. A test can verify that key routes and response codes remain documented:

~~~python
def test_openapi_contains_books_route():
    client = TestClient(create_app())
    schema = client.get("/openapi.json").json()

    assert "/books" in schema["paths"]
    assert "post" in schema["paths"]["/books"]
~~~

Schema tests are useful when clients generate code from the API contract. Keep them focused on stable parts of the schema so incidental formatting does not make them brittle.

## Practice

1. Test a successful request and verify the response body and status.
2. Test a missing resource, an invalid body, and an invalid query parameter.
3. Use an app factory so each test starts with independent data.
4. Override authentication and database dependencies for tests.
5. Use `TestClient` as a context manager to exercise app lifespan.
6. Add one focused OpenAPI check for a route that external clients depend on.

## Main references

- [FastAPI testing documentation](https://fastapi.tiangolo.com/)
- [FastAPI dependency overrides](https://fastapi.tiangolo.com/advanced/testing-dependencies/)
- [FastAPI test client reference](https://fastapi.tiangolo.com/reference/testclient/)
- [HTTPX documentation](https://www.python-httpx.org/)
- [pytest documentation](https://docs.pytest.org/en/stable/)
