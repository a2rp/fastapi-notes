# 03. Path operation routing

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Python environment and the first API](./02-python-environment-and-first-api.md) | [Notes index](../README.md) | [Next: Path, query, and header parameters](./04-path-query-and-header-parameters.md) |

## A path operation joins a method, path, and function

A path operation tells FastAPI what to do when an HTTP method and URL path match. The decorator registers the operation, and the function provides its behavior:

~~~python
from fastapi import FastAPI

app = FastAPI()


@app.get("/books")
async def list_books():
    return [{"id": "1", "title": "Clean Code"}]
~~~

This operation handles `GET /books`. FastAPI calls `list_books()` and serializes its returned list as JSON. The function name helps explain the code but does not determine the URL.

## Use the HTTP method to describe the action

The same resource path can support more than one method:

~~~python
@app.get("/books")
async def list_books():
    return [{"id": "1", "title": "Clean Code"}]


@app.post("/books")
async def create_book():
    return {"id": "2", "title": "The Pragmatic Programmer"}
~~~

`GET /books` reads the collection. `POST /books` creates a new resource. The next chapters add request models, validation, and the correct `201 Created` response for the create operation.

Use the method that matches the operation's behavior. Avoid creating action-shaped paths such as `/getBooks` when the resource path and method already express the meaning.

## Register specific paths before general paths

Starlette checks incoming paths against the registered route list in order. Put a fixed path before a parameterized path when both could match:

~~~python
@app.get("/users/me")
async def read_current_user():
    return {"name": "Current user"}


@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"id": user_id}
~~~

If `/users/{user_id}` is registered first, it can consume the word `me` as a user ID. The same principle applies to catch-all paths: add broad patterns only after more specific routes.

## Keep handlers focused

A path operation can call a service or repository and translate its result into an HTTP response. Keep route code focused on the request and response instead of putting all application logic into one function.

~~~python
from fastapi import HTTPException


books = {
    "1": {"id": "1", "title": "Clean Code"},
    "2": {"id": "2", "title": "The Pragmatic Programmer"},
}


@app.get("/books/{book_id}")
async def read_book(book_id: str):
    book = books.get(book_id)

    if book is None:
        raise HTTPException(
            status_code=404,
            detail="Book not found",
        )

    return book
~~~

This example uses an in-memory dictionary to keep routing visible. Its data disappears when the process restarts. A later chapter moves persistence into a database layer.

## Describe an operation in the API documentation

FastAPI reads metadata from the decorator and adds it to the generated OpenAPI document:

~~~python
@app.get(
    "/books",
    tags=["books"],
    summary="List books",
    description="Return the books available in the reading list.",
    response_description="The current collection of books.",
)
async def list_books():
    return [{"id": "1", "title": "Clean Code"}]
~~~

Tags group related operations in the interactive documentation. A summary and description help a client understand the operation without reading its implementation.

A health route can be excluded from the generated schema when it is operational plumbing rather than part of the public API:

~~~python
@app.get("/health", include_in_schema=False)
async def health_check():
    return {"status": "ok"}
~~~

Excluding a route hides it from the OpenAPI schema. It does not protect the route or require authentication.

## Split larger APIs with `APIRouter`

`APIRouter` groups related path operations. Put a router in a feature module and include it in the main application:

~~~text
app/
|-- __init__.py
|-- main.py
`-- routers/
    |-- __init__.py
    `-- books.py
~~~

In `app/routers/books.py`:

~~~python
from fastapi import APIRouter, HTTPException

router = APIRouter(
    prefix="/books",
    tags=["books"],
)

books = {
    "1": {"id": "1", "title": "Clean Code"},
    "2": {"id": "2", "title": "The Pragmatic Programmer"},
}


@router.get("/")
async def list_books():
    return list(books.values())


@router.get("/{book_id}")
async def read_book(book_id: str):
    book = books.get(book_id)

    if book is None:
        raise HTTPException(status_code=404, detail="Book not found")

    return book
~~~

In `app/main.py`:

~~~python
from fastapi import FastAPI

from app.routers import books

app = FastAPI(title="Reading List API")
app.include_router(books.router, prefix="/api/v1")
~~~

The router prefix and the `include_router()` prefix combine with the path declared on the operation. These routes are available as `GET /api/v1/books/` and `GET /api/v1/books/{book_id}`.

Choose one trailing-slash style for a collection route and use it consistently. FastAPI can redirect between slash variants by default, but clients and tests should use the documented canonical path.

## Think about route ownership

A path should have one clear owner. If unrelated modules register the same method and path, the behavior becomes difficult to understand and the generated schema may contain conflicts.

Before adding an operation, check the route table for:

- The same method and path registered more than once.
- A broad parameter or catch-all route registered before a specific route.
- A router prefix duplicated in both the router and the path operation.
- A public route that should instead require a dependency or permission check.

Routing answers which function handles a request. Request validation, authentication, authorization, and database access remain separate responsibilities.

## Try the route

Start the app and request both paths:

~~~sh
curl -i http://127.0.0.1:8000/books
curl -i http://127.0.0.1:8000/books/1
~~~

The `-i` option displays response status and headers along with the body. Compare the result with the method and path registered by the decorator.

## Main references

- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [FastAPI `APIRouter` reference](https://fastapi.tiangolo.com/reference/apirouter/)
- [FastAPI application reference](https://fastapi.tiangolo.com/reference/fastapi/)
- [Starlette routing and route priority](https://www.starlette.io/routing/)