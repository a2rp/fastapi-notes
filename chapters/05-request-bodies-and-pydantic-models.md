# 05. Request bodies and Pydantic models

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Path, query, and header parameters](./04-path-query-and-header-parameters.md) | [Notes index](../README.md) | [Next: Dependencies and reusable logic](./06-dependencies-and-reusable-logic.md) |

## A request body carries structured data

Clients usually send JSON in the body of a `POST`, `PUT`, or `PATCH` request. A Pydantic model describes the expected shape. FastAPI reads the JSON, validates it against that model, and passes a typed model instance to the endpoint.

This keeps input rules close to the endpoint signature. Invalid JSON or values that do not meet the model rules are rejected before the endpoint function runs.

## Define a request model

Start with a class that inherits from `BaseModel`. Each annotated attribute becomes a field:

~~~python
from pydantic import BaseModel


class BookCreate(BaseModel):
    title: str
    author: str
    year: int
    price: float
~~~

Now use the model as an endpoint parameter:

~~~python
from fastapi import FastAPI

app = FastAPI()


@app.post("/books")
async def create_book(book: BookCreate):
    return {
        "title": book.title,
        "author": book.author,
        "year": book.year,
        "price": book.price,
    }
~~~

Send a JSON object whose keys match the field names. FastAPI gives the endpoint a `BookCreate` instance, so fields are available with normal attribute access such as `book.title`.

Create a file named `book.json` with this content:

~~~json
{
  "title": "The Hobbit",
  "author": "J. R. R. Tolkien",
  "year": 1937,
  "price": 12.5
}
~~~

Then send it with curl:

~~~sh
curl.exe -i http://127.0.0.1:8000/books -H "Content-Type: application/json" --data-binary @book.json
~~~

## Required values, defaults, and nullable values

A field without a default is required. A default makes it optional in the incoming JSON:

~~~python
from pydantic import BaseModel, Field


class BookCreate(BaseModel):
    title: str
    year: int
    language: str = "en"
    tags: list[str] = Field(default_factory=list)
~~~

Here, `title` and `year` must be present. `language` defaults to `en`, and each model instance gets its own empty `tags` list. A default factory is the safe way to create mutable defaults.

Optional and nullable describe different things. `str | None` allows the value `null`, but the field is still required if it has no default:

~~~python
class BookCreate(BaseModel):
    title: str
    subtitle: str | None
~~~

This body must include `subtitle`, whose value may be a string or `null`. To make it optional as well, give it a default:

~~~python
class BookCreate(BaseModel):
    title: str
    subtitle: str | None = None
~~~

Now a client may omit `subtitle` or send it as `null`. This distinction is useful when an update operation must tell the difference between a missing field and a field the client explicitly clears.

## Add field constraints

Use `Field()` to validate limits and add descriptions to the generated API schema:

~~~python
from pydantic import BaseModel, Field


class BookCreate(BaseModel):
    title: str = Field(min_length=1, max_length=160)
    year: int = Field(ge=1, le=2100)
    price: float = Field(gt=0, description="Price in the selected currency")
    tags: list[str] = Field(default_factory=list, max_length=10)
~~~

Common string rules include `min_length`, `max_length`, and `pattern`. Numeric rules include `gt`, `ge`, `lt`, and `le`. Constraints are checked for every request, even when a caller bypasses the interactive API docs.

For money, use a decimal type if exact decimal arithmetic is required. A binary floating-point value can have small representation differences.

## Validate nested objects and lists

Models can contain other models and typed lists. This lets the request shape reflect the JSON structure:

~~~python
from pydantic import BaseModel, Field


class Author(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    website: str | None = None


class Review(BaseModel):
    rating: int = Field(ge=1, le=5)
    comment: str = Field(min_length=1, max_length=500)


class BookCreate(BaseModel):
    title: str = Field(min_length=1, max_length=160)
    year: int = Field(ge=1, le=2100)
    author: Author
    reviews: list[Review] = Field(default_factory=list)
~~~

An accepted body can look like this:

~~~json
{
  "title": "The Hobbit",
  "year": 1937,
  "author": {
    "name": "J. R. R. Tolkien",
    "website": null
  },
  "reviews": [
    {
      "rating": 5,
      "comment": "A memorable adventure"
    }
  ]
}
~~~

Pydantic checks the nested author, every review, and each field rule. If one nested value has the wrong type or falls outside a limit, FastAPI returns a request validation error and does not call the endpoint.

## Choose how to handle unknown fields

By default, Pydantic models ignore fields that are not declared. For request bodies where misspelled or unexpected keys should be rejected, configure the model to forbid extra data:

~~~python
from pydantic import BaseModel, ConfigDict


class BookCreate(BaseModel):
    model_config = ConfigDict(extra="forbid")

    title: str
    year: int
~~~

With this setting, a body containing `titel` instead of `title` reports both the missing required field and the extra field. Choose the policy to match the API contract and apply it consistently.

## Understand the body shape

When an endpoint has one Pydantic model parameter, FastAPI expects the model fields at the top level of the JSON object:

~~~json
{
  "title": "The Hobbit",
  "year": 1937
}
~~~

A plain scalar parameter normally comes from the query string. Use `Body()` to declare a scalar body value explicitly. Use `embed=True` when the JSON should wrap the value under its parameter name:

~~~python
from typing import Annotated

from fastapi import Body, FastAPI

app = FastAPI()


@app.put("/settings/visibility")
async def set_visibility(
    enabled: Annotated[bool, Body(embed=True)],
):
    return {"enabled": enabled}
~~~

This endpoint expects `{"enabled": true}`. Without embedding, an explicit scalar body would be the JSON value `true` itself.

When an endpoint needs more than one body value, declare each parameter with `Body()` so the JSON has a clear key for each value:

~~~python
from typing import Annotated

from fastapi import Body


@app.post("/books/with-discount")
async def create_book_with_discount(
    book: BookCreate,
    discount_percent: Annotated[int, Body(ge=0, le=80)],
):
    return {
        "book": book,
        "discount_percent": discount_percent,
    }
~~~

The body contains both keys, for example `{"book":{"title":"The Hobbit","year":1937},"discount_percent":10}`. Keep request shapes predictable and document them through clear model and field names.

## Use separate models for create and partial update

A create request usually requires the fields needed to make a valid record. A partial update allows each editable field to be omitted:

~~~python
from pydantic import BaseModel, Field


class BookCreate(BaseModel):
    title: str = Field(min_length=1, max_length=160)
    year: int = Field(ge=1, le=2100)
    language: str = "en"


class BookUpdate(BaseModel):
    title: str | None = Field(default=None, min_length=1, max_length=160)
    year: int | None = Field(default=None, ge=1, le=2100)
    language: str | None = None


@app.patch("/books/{book_id}")
async def update_book(book_id: int, changes: BookUpdate):
    supplied_fields = changes.model_dump(exclude_unset=True)
    return {
        "book_id": book_id,
        "changes": supplied_fields,
    }
~~~

`exclude_unset=True` includes only fields sent by the client. If the body is `{"language":null}`, the update data keeps `language` with a `None` value. If `language` is omitted, it is absent from the update data. This lets application code distinguish clearing a value from leaving it unchanged.

Do not use the same input model as a public response model when it contains private fields. Define response shapes separately; the next chapter covers response models.

## Read the validation response

Try requests with a valid body, then remove `title`, send a string for `year`, use a negative review rating, or add an unexpected key after setting `extra="forbid"`. FastAPI returns a `422` response with a `detail` array. Each error identifies a location, a message, an error type, and the rejected input when available.

The exact error text can vary by framework and validation library version. Clients should rely on the documented response shape and status, not on matching a full English message.

The generated OpenAPI schema includes the model fields, types, required values, descriptions, and constraints. This makes the request contract visible in the interactive docs and available to client tooling.

## Practice

1. Add a `BookCreate` model with a required title, an optional summary, a year from 1 through 2100, and a list of tags.
2. Add an `Author` model and require a nested author object in the book body.
3. Try one valid request and at least three invalid requests. Record which rule rejects each body.
4. Add `extra="forbid"`, send a misspelled field name, and inspect the error details.
5. Create a partial update model and compare an omitted value with an explicit `null` using `model_dump(exclude_unset=True)`.

## Main references

- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [Pydantic models](https://docs.pydantic.dev/latest/concepts/models/)
- [Pydantic fields](https://docs.pydantic.dev/latest/concepts/fields/)
- [Python typing documentation](https://docs.python.org/3/library/typing.html)
