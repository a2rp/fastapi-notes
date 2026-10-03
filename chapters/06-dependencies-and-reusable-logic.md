# 06. Dependencies and reusable logic

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Request bodies and Pydantic models](./05-request-bodies-and-pydantic-models.md) | [Notes index](../README.md) | [Next: Response models and status codes](./07-response-models-and-status-codes.md) |

## A dependency provides a value to an endpoint

A dependency is a callable that FastAPI runs before an endpoint and whose result it can pass into that endpoint. Declare it with `Depends()`. FastAPI inspects its parameters, resolves any dependencies it needs, and supplies the result to the route function.

Dependencies are useful for shared query parsing, authenticated users, database sessions, configuration, and other request-scoped work. They keep endpoint functions focused on the operation itself.

## Share common query parameters

For a small example, extract search and pagination values into a function:

~~~python
from typing import Annotated

from fastapi import Depends, FastAPI

app = FastAPI()


async def get_pagination(
    q: str | None = None,
    skip: int = 0,
    limit: int = 20,
):
    return {"q": q, "skip": skip, "limit": limit}


Pagination = Annotated[dict, Depends(get_pagination)]


@app.get("/books")
async def list_books(pagination: Pagination):
    return pagination


@app.get("/authors")
async def list_authors(pagination: Pagination):
    return pagination
~~~

FastAPI reads the dependency parameters from the query string. A request such as `/books?q=python&skip=20&limit=10` runs `get_pagination` and passes its returned dictionary as `pagination`.

The `Annotated` form keeps the dependency declaration next to the type. It also keeps the plain Python type visible to editors and static checkers.

## Validate shared values in the dependency

Dependencies can use the same parameter declarations as path operations. Add constraints before returning the values:

~~~python
from typing import Annotated

from fastapi import Depends, Query


async def get_pagination(
    q: Annotated[str | None, Query(max_length=80)] = None,
    skip: Annotated[int, Query(ge=0)] = 0,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
):
    return {"q": q, "skip": skip, "limit": limit}
~~~

FastAPI validates these values before calling the endpoint. A negative `skip` or a `limit` above 100 receives the usual request validation response. The rules and generated API documentation stay consistent anywhere this dependency is used.

## Use a class as a dependency

A class can collect related values into a typed object. FastAPI calls the class with the resolved parameters, just as it calls a function:

~~~python
from typing import Annotated

from fastapi import Depends, FastAPI, Query

app = FastAPI()


class Pagination:
    def __init__(
        self,
        skip: Annotated[int, Query(ge=0)] = 0,
        limit: Annotated[int, Query(ge=1, le=100)] = 20,
    ):
        self.skip = skip
        self.limit = limit


PaginationParams = Annotated[Pagination, Depends(Pagination)]


@app.get("/books")
async def list_books(page: PaginationParams):
    return {"skip": page.skip, "limit": page.limit}
~~~

Use a class when the result naturally groups state or related options. A function is simpler when it returns a small value or performs a single check.

## Build a dependency tree

A dependency may depend on another dependency. FastAPI resolves the graph from the leaves upward and passes each result to the callable that requested it:

~~~python
from typing import Annotated

from fastapi import Depends, FastAPI, Header, HTTPException

app = FastAPI()


async def get_token(
    authorization: Annotated[str | None, Header()] = None,
):
    if authorization is None or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Bearer token required")
    return authorization.removeprefix("Bearer ").strip()


async def get_current_user(
    token: Annotated[str, Depends(get_token)],
):
    if token != "reader-token":
        raise HTTPException(status_code=401, detail="Invalid token")
    return {"id": 7, "name": "Sam"}


CurrentUser = Annotated[dict, Depends(get_current_user)]


@app.get("/profile")
async def read_profile(user: CurrentUser):
    return user
~~~

This small example demonstrates dependency flow, not a production authentication design. Real applications should verify signed credentials or use a trusted identity provider. The authentication chapter covers those choices.

## Run a dependency without using its return value

Use `dependencies=[Depends(...)]` when an endpoint needs a check to run but does not need its result:

~~~python
from typing import Annotated

from fastapi import Depends, FastAPI, Header, HTTPException

app = FastAPI()


async def require_internal_key(
    internal_key: Annotated[str | None, Header(alias="X-Internal-Key")] = None,
):
    if internal_key != "local-development-key":
        raise HTTPException(status_code=403, detail="Internal key required")


@app.get("/internal/health", dependencies=[Depends(require_internal_key)])
async def internal_health():
    return {"status": "ok"}
~~~

Use this for checks that either complete successfully or raise an exception, such as verifying a required permission. Do not treat a fixed example key as secure production authentication.

## Use `yield` to clean up a resource

A dependency can yield a value and run cleanup code after the request uses it. This pattern is common for resources that must always be closed:

~~~python
from collections.abc import Iterator
from typing import Annotated

from fastapi import Depends, FastAPI

app = FastAPI()


class DemoResource:
    def close(self):
        print("resource closed")


def get_resource() -> Iterator[DemoResource]:
    resource = DemoResource()
    try:
        yield resource
    finally:
        resource.close()


Resource = Annotated[DemoResource, Depends(get_resource)]


@app.get("/resource")
async def use_resource(resource: Resource):
    return {"resource": type(resource).__name__}
~~~

Code before `yield` prepares the resource. The yielded object is injected into the endpoint. The `finally` block runs during dependency cleanup, including when the endpoint raises an exception. Use a context manager or `try` and `finally` so cleanup is not skipped.

For an asynchronous resource, use an async generator and `async with` or `await` its asynchronous close method. Database session examples appear in the persistence chapter.

## Understand dependency caching

FastAPI normally caches a dependency result for one request. If the same dependency is needed more than once in that request, it is called once and its result is reused. This is helpful when several layers share one session or one authenticated user.

Set `use_cache=False` only when a repeated call is intentional:

~~~python
from typing import Annotated

from fastapi import Depends


async def create_request_value():
    return object()


FreshValue = Annotated[object, Depends(create_request_value, use_cache=False)]
~~~

Disabling caching can repeat network calls, database work, or other side effects. Keep the default unless the endpoint needs distinct values from repeated executions.

## Override a dependency in a test

The app exposes `dependency_overrides`, a mapping from the original dependency callable to a replacement. This makes it possible to test a route without its real external service:

~~~python
from fastapi.testclient import TestClient


async def fake_current_user():
    return {"id": 99, "name": "Test Reader"}


original_overrides = app.dependency_overrides.copy()
try:
    app.dependency_overrides[get_current_user] = fake_current_user
    response = TestClient(app).get("/profile")
    assert response.status_code == 200
    assert response.json() == {"id": 99, "name": "Test Reader"}
finally:
    app.dependency_overrides = original_overrides
~~~

Restore the mapping after each test so one test cannot change another test result. The testing chapter expands this pattern with fixtures and application factories.

## Practice

1. Add a shared pagination dependency and use it in two list endpoints.
2. Add validation limits for `skip` and `limit`, then send invalid values.
3. Convert a small dependency function into a class and compare which form reads more clearly.
4. Create a resource dependency with `yield` and confirm its cleanup runs after success and after an exception.
5. Override an external-service dependency with a deterministic fake in a test.

## Main references

- [FastAPI `Depends` reference](https://fastapi.tiangolo.com/reference/dependencies/)
- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [Python type annotations](https://docs.python.org/3/library/typing.html)
- [Python context managers](https://docs.python.org/3/library/contextlib.html)
