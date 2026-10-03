# 07. Response models and status codes

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Dependencies and reusable logic](./06-dependencies-and-reusable-logic.md) | [Notes index](../README.md) | [Next: Errors and exception handling](./08-errors-and-exception-handling.md) |

## A response model defines the public shape

A response model describes the data an endpoint is allowed to return. FastAPI uses it to document the response, convert values to JSON, and filter out fields that are not part of the public shape.

Use separate models for incoming data and outgoing data. A database record may contain fields such as an internal identifier, a password hash, or private notes that must not be sent to a client.

## Keep private fields out of the response

In this example, the input has a password, the internal record also has a password hash, and the public response contains neither:

~~~python
from pydantic import BaseModel


class AccountCreate(BaseModel):
    email: str
    password: str


class AccountRecord(BaseModel):
    id: int
    email: str
    password_hash: str


class AccountPublic(BaseModel):
    id: int
    email: str
~~~

Declare the public response model on the operation:

~~~python
from fastapi import FastAPI, status

app = FastAPI()


@app.post("/accounts", response_model=AccountPublic, status_code=status.HTTP_201_CREATED)
async def create_account(account: AccountCreate):
    saved_record = AccountRecord(
        id=1,
        email=account.email,
        password_hash="stored-hash-value",
    )
    return saved_record
~~~

The response JSON contains only `id` and `email`. Returning a record with additional fields does not add those fields to the output. This is an important boundary, but never store a plain-text password as shown in an incoming example; password storage belongs to the security chapter.

## Return a list or a nested response

A response model can be a single Pydantic type or a collection of types:

~~~python
from pydantic import BaseModel


class BookPublic(BaseModel):
    id: int
    title: str
    author: str


@app.get("/books", response_model=list[BookPublic])
async def list_books():
    return [
        {"id": 1, "title": "The Hobbit", "author": "J. R. R. Tolkien"},
        {"id": 2, "title": "A Wizard of Earthsea", "author": "Ursula K. Le Guin"},
    ]
~~~

FastAPI checks each returned item against `BookPublic`. If the route returns data that does not satisfy its declared output shape, that is a server-side programming error and results in a server error. It should be corrected in the application, not presented as invalid client input.

## Map an ORM object to a response model

Database objects are not always dictionaries. With Pydantic v2, `from_attributes=True` allows a response model to read values from object attributes:

~~~python
from pydantic import BaseModel, ConfigDict


class BookPublic(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str
    author: str


def to_public_book(database_row):
    return BookPublic.model_validate(database_row)
~~~

The object must provide attributes with the names expected by the model. Database session setup and persistence are covered in the database chapter.

## Control optional output fields deliberately

FastAPI response models support options such as `response_model_exclude_none=True` and `response_model_exclude_unset=True`. These can omit `None` values or fields that were not explicitly set.

~~~python
@app.get(
    "/profile",
    response_model=ProfilePublic,
    response_model_exclude_none=True,
)
async def read_profile():
    return {"id": 4, "display_name": "Asha", "bio": None}
~~~

For a stable public API, prefer defining the desired output shape directly. Broad include and exclude rules can make the generated schema disagree with the actual JSON, so keep such options narrow and test the result.

## Set the correct success status

FastAPI defaults to `200 OK` for a successful operation. A creation endpoint usually responds with `201 Created`:

~~~python
from fastapi import FastAPI, status

app = FastAPI()


@app.post("/books", status_code=status.HTTP_201_CREATED)
async def create_book():
    return {"id": 1, "title": "The Hobbit"}
~~~

Using a named constant from `fastapi.status` makes the intent easier to read. The status code is also included in the generated API description.

Common success codes include:

- `200 OK` for a successful read or update with a response body.
- `201 Created` when a new resource has been created.
- `202 Accepted` when work has been accepted but is not finished.
- `204 No Content` when the operation succeeds and returns no response body.

Use a code that matches what the endpoint actually did. Returning `202` does not itself queue work, and returning `201` does not create a database record.

## Choose a status for an empty response

A delete operation can return `204 No Content` after removing a resource:

~~~python
from fastapi import FastAPI, Response, status

app = FastAPI()
book_ids = {1, 2, 3}


@app.delete("/books/{book_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_book(book_id: int):
    book_ids.discard(book_id)
    return Response(status_code=status.HTTP_204_NO_CONTENT)
~~~

A `204` response has no JSON body. If the client needs a message, deleted identifier, or other result, return a body with an appropriate status such as `200` instead.

## Change a status for one result

An operation can have a normal response model and choose a different status for a particular result by receiving a temporary `Response` parameter:

~~~python
from typing import Annotated

from fastapi import FastAPI, Response, status

app = FastAPI()
settings: set[str] = set()


@app.put("/settings/{key}", status_code=status.HTTP_200_OK)
async def set_setting(key: str, response: Response):
    created = key not in settings
    settings.add(key)
    if created:
        response.status_code = status.HTTP_201_CREATED
    return {"key": key, "saved": True}
~~~

FastAPI keeps normal response conversion and filtering when you set status on the injected response object. Returning a `Response` instance directly is a different choice and bypasses automatic response model processing.

## Understand direct response objects

Return a response class directly only when you need to control the actual response body or media type:

~~~python
from fastapi import FastAPI, Response

app = FastAPI()


@app.get("/legacy-feed")
async def legacy_feed():
    xml = "<feed><entry>One item</entry></feed>"
    return Response(content=xml, media_type="application/xml")
~~~

A directly returned response is sent as provided. FastAPI does not validate or filter its content through a response model automatically. When you need ordinary JSON with a public schema, return data and declare `response_model` instead.

## Practice

1. Create separate input, stored-record, and public response models for a user or book.
2. Return a record containing a private field and verify that the public response omits it.
3. Add a list response model and make one returned item invalid to see the server-side failure.
4. Give a create endpoint the `201 Created` status and a delete endpoint `204 No Content`.
5. Compare setting `response.status_code` with returning a `Response` instance directly.

## Main references

- [FastAPI operation and response model reference](https://fastapi.tiangolo.com/reference/fastapi/)
- [FastAPI response status codes](https://fastapi.tiangolo.com/advanced/response-change-status-code/)
- [FastAPI direct responses](https://fastapi.tiangolo.com/advanced/response-directly/)
- [Pydantic models](https://docs.pydantic.dev/latest/concepts/models/)
