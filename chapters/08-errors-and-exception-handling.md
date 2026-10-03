# 08. Errors and exception handling

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Response models and status codes](./07-response-models-and-status-codes.md) | [Notes index](../README.md) | [Next: Forms, files, and static resources](./09-forms-files-and-static-resources.md) |

## Report a failed request with an HTTP status

Use an HTTP status code that tells the client what happened. A `4xx` status usually means the request cannot be completed as sent. A `5xx` status means the server could not complete a request that may otherwise be valid.

FastAPI raises `HTTPException` to stop the current request and send an error response. Raise it from a route, dependency, or helper when the application has a known reason to reject the operation.

## Raise `HTTPException` for a missing resource

Look up a resource and raise a `404 Not Found` response when it does not exist:

~~~python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()

books = {
    1: {"id": 1, "title": "The Hobbit"},
}


@app.get("/books/{book_id}")
async def read_book(book_id: int):
    book = books.get(book_id)
    if book is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Book not found",
        )
    return book
~~~

After the exception is raised, the rest of the endpoint does not run. FastAPI sends a JSON response with a `detail` field and the selected status code.

## Use status codes consistently

Choose the status that best matches the reason for rejection:

- `400 Bad Request` for a request the application cannot accept because of a general client-side problem.
- `401 Unauthorized` when valid authentication is required. Include the appropriate authentication challenge when the scheme requires one.
- `403 Forbidden` when the caller is known but is not allowed to perform the operation.
- `404 Not Found` when a resource does not exist or the API intentionally hides whether it exists.
- `409 Conflict` when the requested change conflicts with the current resource state.
- `422 Unprocessable Content` when the request structure or field values fail validation.

Do not use `401` and `403` interchangeably. Authentication answers who the caller is; authorization answers what that caller may do.

## Return structured error details

The `detail` value can contain JSON-compatible data. A stable error code can help clients react without parsing a sentence:

~~~python
from fastapi import HTTPException, status


def reject_duplicate_email(email: str):
    raise HTTPException(
        status_code=status.HTTP_409_CONFLICT,
        detail={
            "code": "email_already_registered",
            "message": "An account already uses this email address.",
            "email": email,
        },
    )
~~~

Only include data the caller is allowed to see. A predictable error code is more stable for clients than the exact wording of a message.

## Add a response header to an error

An exception can include headers. This is useful when the response must explain a retry or authentication condition:

~~~python
from fastapi import HTTPException


def require_api_key(api_key: str | None):
    if api_key is None:
        raise HTTPException(
            status_code=401,
            detail="API key required",
            headers={"WWW-Authenticate": "Bearer"},
        )
~~~

Use the challenge that matches the authentication scheme you have implemented. Do not advertise a scheme that the API does not support.

## Handle a domain-specific exception centrally

Raise a plain application exception where the business rule is checked, then translate it to HTTP at the app boundary:

~~~python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()


class InventoryConflict(Exception):
    def __init__(self, item_id: int):
        self.item_id = item_id


@app.exception_handler(InventoryConflict)
async def inventory_conflict_handler(
    request: Request,
    exc: InventoryConflict,
):
    return JSONResponse(
        status_code=409,
        content={
            "code": "inventory_conflict",
            "message": "The requested stock change is not available.",
            "item_id": exc.item_id,
        },
    )


@app.post("/inventory/{item_id}/reserve")
async def reserve_item(item_id: int):
    raise InventoryConflict(item_id)
~~~

An exception handler is a good fit when the same domain error can happen in many routes. Keep the exception independent of HTTP where practical, then map it to a response once.

## Know the built-in request validation error

FastAPI validates path values, query values, headers, and Pydantic request bodies before calling the endpoint. Invalid input normally returns `422 Unprocessable Content` with a `detail` array.

Each error identifies its location, an error type, and a readable message. For example, a non-integer value in `/books/not-a-number` points to the path field. The exact message text can change with library versions, so clients should not rely on parsing it.

The framework already provides a useful response. Keep it unless the application has a concrete reason to use a different format.

## Customize validation errors only for a clear API contract

An application can replace the default validation handler. This example keeps the location and error type while omitting rejected input values that may contain sensitive data:

~~~python
from fastapi import FastAPI, Request
from fastapi.encoders import jsonable_encoder
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

app = FastAPI()


@app.exception_handler(RequestValidationError)
async def validation_error_handler(
    request: Request,
    exc: RequestValidationError,
):
    errors = [
        {
            "location": list(error["loc"]),
            "code": error["type"],
            "message": error["msg"],
        }
        for error in exc.errors()
    ]
    return JSONResponse(
        status_code=422,
        content=jsonable_encoder({"errors": errors}),
    )
~~~

Customizing the format changes what every client receives, so document and test the new contract. Avoid returning the raw request body by default. It may contain passwords, tokens, or personal data.

## Catch the HTTP exception base class when customizing handlers

FastAPI `HTTPException` extends Starlette `HTTPException`. If a custom handler must also catch exceptions raised by Starlette or its extensions, register it for the Starlette base class:

~~~python
from fastapi import FastAPI
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

app = FastAPI()


@app.exception_handler(StarletteHTTPException)
async def http_error_handler(
    request,
    exc: StarletteHTTPException,
):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.detail},
        headers=exc.headers,
    )
~~~

When overriding a built-in handler, consider calling FastAPI original handler if only logging or adding one shared behavior is needed. Replacing the full response shape affects all routes and framework-generated errors.

## Let unexpected failures remain server errors

Do not catch every exception and return `200 OK` or expose an exception traceback to a client. Unexpected errors should remain `500 Internal Server Error`; log useful context on the server and return a safe message.

Logs should help locate a failure without copying secrets or full request bodies. Include a request or trace identifier when available so one client report can be matched to server logs.

## Practice

1. Return `404` when a requested book is absent.
2. Add a stable error code for a duplicate resource and choose a suitable status.
3. Raise a domain exception from a helper and map it with an app-level handler.
4. Send invalid path and body values and inspect the default `422` response.
5. Customize validation output without returning raw user input, then test the revised error contract.

## Main references

- [FastAPI exception reference](https://fastapi.tiangolo.com/reference/exceptions/)
- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [Starlette exceptions](https://www.starlette.io/exceptions/)
- [HTTP status code registry](https://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml)
