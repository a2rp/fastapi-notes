# 04. Path, query, and header parameters

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Path operation routing](./03-path-operation-routing.md) | [Notes index](../README.md) | [Next: Request bodies and Pydantic models](./05-request-bodies-and-pydantic-models.md) |

## FastAPI identifies where simple values come from

FastAPI matches values in a path template to path parameters. Other simple function parameters are treated as query parameters unless you explicitly declare another source. These examples use Python 3.10+ type syntax such as `str | None` and built-in generic types.

For example, in `/books/{book_id}?language=en`, `book_id` comes from the path and `language` comes from the query string. The URL carries text, then FastAPI converts and validates each value using its Python type annotation.

## Validate a path parameter

Use a type annotation to convert the value. Here, a positive integer is required:

~~~python
from typing import Annotated

from fastapi import FastAPI, Path

app = FastAPI()


@app.get("/books/{book_id}")
async def read_book(book_id: Annotated[int, Path(gt=0)]):
    return {"book_id": book_id}
~~~

A request to `/books/12` gives the function the integer `12`. A request to `/books/abc`, or a number that does not satisfy `gt=0`, fails validation before the function runs. FastAPI returns a `422 Unprocessable Entity` response with details about the invalid input.

Path parameters are always required because they are part of the URL path. `Path()` adds constraints and documentation metadata; it does not make a path segment optional.

## Add optional and required query parameters

A parameter with a default is optional. A parameter with no default is required:

~~~python
from typing import Annotated

from fastapi import FastAPI, Query

app = FastAPI()


@app.get("/books")
async def list_books(
    q: Annotated[str | None, Query(min_length=2, max_length=50)] = None,
    page: Annotated[int, Query(ge=1)] = 1,
    page_size: Annotated[int, Query(ge=1, le=100)] = 20,
):
    return {
        "q": q,
        "page": page,
        "page_size": page_size,
    }


@app.get("/search")
async def search_books(
    q: Annotated[str, Query(min_length=3, max_length=80)],
):
    return {"query": q}
~~~

The `q` value on `/books` is optional because its default is `None`. The `q` value on `/search` is required because it has no default.

Numeric constraints include `gt`, `ge`, `lt`, and `le`, meaning greater than, greater than or equal, less than, and less than or equal. String constraints include `min_length`, `max_length`, and `pattern`.

## Use a different public query name

A Python variable name does not have to match the name clients send. Use `alias` when the public query parameter contains punctuation or when the external name is part of an existing API contract:

~~~python
from typing import Annotated

from fastapi import Query


@app.get("/search")
async def search_books(
    search_text: Annotated[str | None, Query(alias="item-query")] = None,
):
    return {"query": search_text}
~~~

A request can use `GET /search?item-query=python`. Inside the function, the value is called `search_text`.

## Accept a query parameter more than once

Use a list annotation and `Query()` to accept repeated values:

~~~python
from typing import Annotated

from fastapi import Query


@app.get("/books/by-tag")
async def list_books_by_tag(
    tag: Annotated[list[str] | None, Query()] = None,
):
    return {"tags": tag or []}
~~~

A request such as `/books?tag=python&tag=fastapi` gives the function `["python", "fastapi"]`. `Query()` makes it clear that the list comes from the query string rather than a request body.

## Read a header parameter

Use `Header()` to declare a request header:

~~~python
from typing import Annotated

from fastapi import Header


@app.get("/client")
async def read_client(
    user_agent: Annotated[str | None, Header()] = None,
):
    return {"user_agent": user_agent}
~~~

HTTP header names are case-insensitive and usually use hyphens. FastAPI converts underscores in the Python parameter name to hyphens by default, so `user_agent` reads the `User-Agent` header.

For an explicit header name, use an alias:

~~~python
from typing import Annotated

from fastapi import Header


@app.get("/request-info")
async def read_request_info(
    request_id: Annotated[str | None, Header(alias="X-Request-ID")] = None,
):
    return {"request_id": request_id}
~~~

Headers are still untrusted request data. A client can choose its own `User-Agent` or `X-Request-ID`. Do not treat a custom header as proof of identity or permission. The authentication chapter covers verified credentials.

## Combine path, query, and header values

FastAPI identifies each value by the path template and parameter declaration. You do not have to group parameters by source:

~~~python
from typing import Annotated
from uuid import UUID

from fastapi import Header, Path, Query


@app.get("/accounts/{account_id}/books")
async def list_account_books(
    account_id: Annotated[UUID, Path()],
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    request_id: Annotated[str | None, Header(alias="X-Request-ID")] = None,
):
    return {
        "account_id": str(account_id),
        "limit": limit,
        "request_id": request_id,
    }
~~~

A URL such as `/accounts/2b4a8ca1-c019-4f4b-b127-c6e67fd1f274/books?limit=25` supplies the UUID and query value. The header is sent separately from the URL.

When a value has a more complex nested structure, use a Pydantic request model in the body. The next chapter covers that case.

## See validation in the generated schema

Parameter types and constraints appear in the generated OpenAPI schema and interactive documentation. A client can see which values are required, accepted, or limited before it sends a request.

This documentation describes the contract, but the server must still validate every incoming request. Never rely on a browser or API client to enforce constraints.

## Try requests from a terminal

~~~sh
curl -i "http://127.0.0.1:8000/books?q=python&page=1&page_size=10"
curl -i "http://127.0.0.1:8000/books/by-tag?tag=python&tag=fastapi"
curl -i "http://127.0.0.1:8000/books/12"
curl -i "http://127.0.0.1:8000/client" -H "User-Agent: notes-client"
curl -i "http://127.0.0.1:8000/request-info" -H "X-Request-ID: request-123"
~~~

Change a valid value to an invalid one, such as `page_size=0`, and inspect the `422` response. This makes the validation behavior visible without adding custom error code.

## Main references

- [FastAPI request parameter reference](https://fastapi.tiangolo.com/reference/parameters/)
- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [Starlette routing](https://www.starlette.io/routing/)
- [Python typing documentation](https://docs.python.org/3/library/typing.html)
- [Python UUID documentation](https://docs.python.org/3/library/uuid.html)