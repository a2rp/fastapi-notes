# 98. All code samples

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Deployment, performance, and operations](./16-deployment-performance-and-operations.md) | [Notes index](../README.md) | [Next: Complete questions and answers](./99-complete-q-and-a.md) |

This appendix gathers every fenced example from the 16 core chapters. Examples remain grouped by chapter and use the same language labels as their source. Some snippets build on names or setup introduced earlier in that chapter.

## 01. FastAPI and the ASGI request lifecycle
[Open the chapter](./01-fastapi-and-asgi.md)

### Example 1: text

~~~text
HTTP client
    |
    v
Uvicorn ASGI server
    |
    v
FastAPI application
    |
    v
matching path operation and dependencies
    |
    v
response validation and serialization
    |
    v
Uvicorn sends the HTTP response to the client
~~~

### Example 2: python

~~~python
from fastapi import FastAPI

app = FastAPI(title="Reading List API")


@app.get("/")
async def read_home():
    return {"message": "Hello, FastAPI!"}
~~~

### Example 3: sh

~~~sh
uvicorn main:app --reload
~~~

### Example 4: python

~~~python
@app.get("/external-status")
async def read_external_status():
    result = await status_client.fetch()
    return {"status": result}
~~~

### Example 5: python

~~~python
@app.get("/legacy-status")
def read_legacy_status():
    result = blocking_status_client.fetch()
    return {"status": result}
~~~


## 02. Python environment and the first API
[Open the chapter](./02-python-environment-and-first-api.md)

### Example 1: sh

~~~sh
python --version
python -m pip --version
~~~

### Example 2: powershell

~~~powershell
py -3 --version
~~~

### Example 3: sh

~~~sh
python -m venv .venv
~~~

### Example 4: powershell

~~~powershell
py -3 -m venv .venv
~~~

### Example 5: powershell

~~~powershell
.\.venv\Scripts\Activate.ps1
~~~

### Example 6: sh

~~~sh
source .venv/bin/activate
~~~

### Example 7: sh

~~~sh
python -m pip install --upgrade pip
python -m pip install "fastapi[standard]"
~~~

### Example 8: text

~~~text
fastapi[standard]
~~~

### Example 9: sh

~~~sh
python -m pip install -r requirements.txt
~~~

### Example 10: text

~~~text
reading-list/
|-- .venv/
|-- main.py
`-- requirements.txt
~~~

### Example 11: python

~~~python
from fastapi import FastAPI

app = FastAPI(title="Reading List API")


@app.get("/")
async def read_home():
    return {"message": "Hello, FastAPI!"}
~~~

### Example 12: sh

~~~sh
python -m uvicorn main:app --reload
~~~

### Example 13: text

~~~text
reading-list/
|-- app/
|   |-- __init__.py
|   `-- main.py
|-- .venv/
`-- requirements.txt
~~~

### Example 14: sh

~~~sh
python -m uvicorn app.main:app --reload
~~~

### Example 15: sh

~~~sh
fastapi dev main.py
~~~

### Example 16: sh

~~~sh
python -m pip show fastapi uvicorn
~~~


## 03. Path operation routing
[Open the chapter](./03-path-operation-routing.md)

### Example 1: python

~~~python
from fastapi import FastAPI

app = FastAPI()


@app.get("/books")
async def list_books():
    return [{"id": "1", "title": "Clean Code"}]
~~~

### Example 2: python

~~~python
@app.get("/books")
async def list_books():
    return [{"id": "1", "title": "Clean Code"}]


@app.post("/books")
async def create_book():
    return {"id": "2", "title": "The Pragmatic Programmer"}
~~~

### Example 3: python

~~~python
@app.get("/users/me")
async def read_current_user():
    return {"name": "Current user"}


@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"id": user_id}
~~~

### Example 4: python

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

### Example 5: python

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

### Example 6: python

~~~python
@app.get("/health", include_in_schema=False)
async def health_check():
    return {"status": "ok"}
~~~

### Example 7: text

~~~text
app/
|-- __init__.py
|-- main.py
`-- routers/
    |-- __init__.py
    `-- books.py
~~~

### Example 8: python

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

### Example 9: python

~~~python
from fastapi import FastAPI

from app.routers import books

app = FastAPI(title="Reading List API")
app.include_router(books.router, prefix="/api/v1")
~~~

### Example 10: sh

~~~sh
curl -i http://127.0.0.1:8000/books
curl -i http://127.0.0.1:8000/books/1
~~~


## 04. Path, query, and header parameters
[Open the chapter](./04-path-query-and-header-parameters.md)

### Example 1: python

~~~python
from typing import Annotated

from fastapi import FastAPI, Path

app = FastAPI()


@app.get("/books/{book_id}")
async def read_book(book_id: Annotated[int, Path(gt=0)]):
    return {"book_id": book_id}
~~~

### Example 2: python

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

### Example 3: python

~~~python
from typing import Annotated

from fastapi import Query


@app.get("/search")
async def search_books(
    search_text: Annotated[str | None, Query(alias="item-query")] = None,
):
    return {"query": search_text}
~~~

### Example 4: python

~~~python
from typing import Annotated

from fastapi import Query


@app.get("/books/by-tag")
async def list_books_by_tag(
    tag: Annotated[list[str] | None, Query()] = None,
):
    return {"tags": tag or []}
~~~

### Example 5: python

~~~python
from typing import Annotated

from fastapi import Header


@app.get("/client")
async def read_client(
    user_agent: Annotated[str | None, Header()] = None,
):
    return {"user_agent": user_agent}
~~~

### Example 6: python

~~~python
from typing import Annotated

from fastapi import Header


@app.get("/request-info")
async def read_request_info(
    request_id: Annotated[str | None, Header(alias="X-Request-ID")] = None,
):
    return {"request_id": request_id}
~~~

### Example 7: python

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

### Example 8: sh

~~~sh
curl -i "http://127.0.0.1:8000/books?q=python&page=1&page_size=10"
curl -i "http://127.0.0.1:8000/books/by-tag?tag=python&tag=fastapi"
curl -i "http://127.0.0.1:8000/books/12"
curl -i "http://127.0.0.1:8000/client" -H "User-Agent: notes-client"
curl -i "http://127.0.0.1:8000/request-info" -H "X-Request-ID: request-123"
~~~


## 05. Request bodies and Pydantic models
[Open the chapter](./05-request-bodies-and-pydantic-models.md)

### Example 1: python

~~~python
from pydantic import BaseModel


class BookCreate(BaseModel):
    title: str
    author: str
    year: int
    price: float
~~~

### Example 2: python

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

### Example 3: json

~~~json
{
  "title": "The Hobbit",
  "author": "J. R. R. Tolkien",
  "year": 1937,
  "price": 12.5
}
~~~

### Example 4: sh

~~~sh
curl.exe -i http://127.0.0.1:8000/books -H "Content-Type: application/json" --data-binary @book.json
~~~

### Example 5: python

~~~python
from pydantic import BaseModel, Field


class BookCreate(BaseModel):
    title: str
    year: int
    language: str = "en"
    tags: list[str] = Field(default_factory=list)
~~~

### Example 6: python

~~~python
class BookCreate(BaseModel):
    title: str
    subtitle: str | None
~~~

### Example 7: python

~~~python
class BookCreate(BaseModel):
    title: str
    subtitle: str | None = None
~~~

### Example 8: python

~~~python
from pydantic import BaseModel, Field


class BookCreate(BaseModel):
    title: str = Field(min_length=1, max_length=160)
    year: int = Field(ge=1, le=2100)
    price: float = Field(gt=0, description="Price in the selected currency")
    tags: list[str] = Field(default_factory=list, max_length=10)
~~~

### Example 9: python

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

### Example 10: json

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

### Example 11: python

~~~python
from pydantic import BaseModel, ConfigDict


class BookCreate(BaseModel):
    model_config = ConfigDict(extra="forbid")

    title: str
    year: int
~~~

### Example 12: json

~~~json
{
  "title": "The Hobbit",
  "year": 1937
}
~~~

### Example 13: python

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

### Example 14: python

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

### Example 15: python

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


## 06. Dependencies and reusable logic
[Open the chapter](./06-dependencies-and-reusable-logic.md)

### Example 1: python

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

### Example 2: python

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

### Example 3: python

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

### Example 4: python

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

### Example 5: python

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

### Example 6: python

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

### Example 7: python

~~~python
from typing import Annotated

from fastapi import Depends


async def create_request_value():
    return object()


FreshValue = Annotated[object, Depends(create_request_value, use_cache=False)]
~~~

### Example 8: python

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


## 07. Response models and status codes
[Open the chapter](./07-response-models-and-status-codes.md)

### Example 1: python

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

### Example 2: python

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

### Example 3: python

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

### Example 4: python

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

### Example 5: python

~~~python
@app.get(
    "/profile",
    response_model=ProfilePublic,
    response_model_exclude_none=True,
)
async def read_profile():
    return {"id": 4, "display_name": "Asha", "bio": None}
~~~

### Example 6: python

~~~python
from fastapi import FastAPI, status

app = FastAPI()


@app.post("/books", status_code=status.HTTP_201_CREATED)
async def create_book():
    return {"id": 1, "title": "The Hobbit"}
~~~

### Example 7: python

~~~python
from fastapi import FastAPI, Response, status

app = FastAPI()
book_ids = {1, 2, 3}


@app.delete("/books/{book_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_book(book_id: int):
    book_ids.discard(book_id)
    return Response(status_code=status.HTTP_204_NO_CONTENT)
~~~

### Example 8: python

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

### Example 9: python

~~~python
from fastapi import FastAPI, Response

app = FastAPI()


@app.get("/legacy-feed")
async def legacy_feed():
    xml = "<feed><entry>One item</entry></feed>"
    return Response(content=xml, media_type="application/xml")
~~~


## 08. Errors and exception handling
[Open the chapter](./08-errors-and-exception-handling.md)

### Example 1: python

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

### Example 2: python

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

### Example 3: python

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

### Example 4: python

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

### Example 5: python

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

### Example 6: python

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


## 09. Forms, files, and static resources
[Open the chapter](./09-forms-files-and-static-resources.md)

### Example 1: sh

~~~sh
python -m pip install python-multipart
~~~

### Example 2: python

~~~python
from typing import Annotated

from fastapi import FastAPI, Form

app = FastAPI()


@app.post("/login")
async def login(
    username: Annotated[str, Form(min_length=1, max_length=80)],
    password: Annotated[str, Form(min_length=1, max_length=200)],
):
    return {"username": username}
~~~

### Example 3: python

~~~python
from typing import Annotated

from fastapi import FastAPI, File

app = FastAPI()


@app.post("/small-uploads")
async def upload_small_file(
    file: Annotated[bytes, File()],
):
    return {"size_bytes": len(file)}
~~~

### Example 4: python

~~~python
from typing import Annotated

from fastapi import FastAPI, File, HTTPException, UploadFile

app = FastAPI()
MAX_FILE_BYTES = 2 * 1024 * 1024


@app.post("/documents")
async def upload_document(
    file: Annotated[UploadFile, File()],
):
    contents = await file.read(MAX_FILE_BYTES + 1)
    if len(contents) > MAX_FILE_BYTES:
        raise HTTPException(status_code=413, detail="File is too large")

    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size_bytes": len(contents),
    }
~~~

### Example 5: python

~~~python
from typing import Annotated

from fastapi import FastAPI, File, Form, UploadFile

app = FastAPI()


@app.post("/profile-photo")
async def upload_profile_photo(
    display_name: Annotated[str, Form(min_length=1, max_length=80)],
    file: Annotated[UploadFile, File()],
):
    return {
        "display_name": display_name,
        "filename": file.filename,
        "content_type": file.content_type,
    }
~~~

### Example 6: sh

~~~sh
curl.exe -i http://127.0.0.1:8000/profile-photo -F "display_name=Sam" -F "file=@avatar.png"
~~~

### Example 7: python

~~~python
from pathlib import Path

from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

app = FastAPI()
Path("static").mkdir(exist_ok=True)
app.mount("/static", StaticFiles(directory="static"), name="static")
~~~

### Example 8: python

~~~python
from pathlib import Path

from fastapi import FastAPI, HTTPException
from fastapi.responses import FileResponse

app = FastAPI()
REPORTS = Path("reports").resolve()


@app.get("/reports/{report_name}")
async def download_report(report_name: str):
    if report_name not in {"monthly.csv", "summary.csv"}:
        raise HTTPException(status_code=404, detail="Report not found")

    path = REPORTS / report_name
    if not path.is_file():
        raise HTTPException(status_code=404, detail="Report not found")

    return FileResponse(
        path,
        media_type="text/csv",
        filename=report_name,
    )
~~~


## 10. Database integration and async I/O
[Open the chapter](./10-database-integration-and-async-io.md)

### Example 1: sh

~~~sh
python -m pip install "sqlalchemy>=2,<3"
~~~

### Example 2: python

~~~python
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(160), index=True)
    author: Mapped[str] = mapped_column(String(120))
~~~

### Example 3: python

~~~python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "sqlite:///./notes.db"

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False},
)
SessionLocal = sessionmaker(
    bind=engine,
    autoflush=False,
    autocommit=False,
)
~~~

### Example 4: python

~~~python
from collections.abc import Generator

from sqlalchemy.orm import Session


def get_db() -> Generator[Session, None, None]:
    with SessionLocal() as session:
        yield session
~~~

### Example 5: python

~~~python
Base.metadata.create_all(bind=engine)
~~~

### Example 6: python

~~~python
from typing import Annotated

from fastapi import Depends, FastAPI, status
from pydantic import BaseModel, ConfigDict, Field
from sqlalchemy import select
from sqlalchemy.orm import Session

app = FastAPI()


class BookCreate(BaseModel):
    title: str = Field(min_length=1, max_length=160)
    author: str = Field(min_length=1, max_length=120)


class BookPublic(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str
    author: str


Database = Annotated[Session, Depends(get_db)]


@app.post("/books", response_model=BookPublic, status_code=status.HTTP_201_CREATED)
def create_book(payload: BookCreate, db: Database):
    book = Book(title=payload.title, author=payload.author)
    db.add(book)
    db.commit()
    db.refresh(book)
    return book


@app.get("/books", response_model=list[BookPublic])
def list_books(db: Database):
    return db.scalars(select(Book).order_by(Book.id)).all()
~~~

### Example 7: python

~~~python
from fastapi import HTTPException
from sqlalchemy.exc import IntegrityError


def insert_unique_record(db: Session, record: Book):
    db.add(record)
    try:
        db.commit()
    except IntegrityError as exc:
        db.rollback()
        raise HTTPException(
            status_code=409,
            detail="A record with these values already exists",
        ) from exc
    db.refresh(record)
    return record
~~~

### Example 8: sh

~~~sh
python -m pip install aiosqlite
~~~

### Example 9: python

~~~python
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine

async_engine = create_async_engine(
    "sqlite+aiosqlite:///./async-notes.db",
    echo=False,
)
AsyncSessionLocal = async_sessionmaker(
    bind=async_engine,
    expire_on_commit=False,
)
~~~

### Example 10: python

~~~python
async with async_engine.begin() as connection:
    await connection.run_sync(Base.metadata.create_all)
~~~

### Example 11: python

~~~python
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import AsyncSession


async def get_async_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
~~~

### Example 12: python

~~~python
from typing import Annotated

from fastapi import Depends
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

AsyncDatabase = Annotated[AsyncSession, Depends(get_async_db)]


@app.get("/async-books", response_model=list[BookPublic])
async def list_books_async(db: AsyncDatabase):
    result = await db.scalars(select(Book).order_by(Book.id))
    return result.all()
~~~

### Example 13: python

~~~python
from contextlib import asynccontextmanager

from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    await async_engine.dispose()


app = FastAPI(lifespan=lifespan)
~~~


## 11. Validation, security, and CORS
[Open the chapter](./11-validation-security-and-cors.md)

### Example 1: python

~~~python
from pydantic import BaseModel, Field, field_validator


class ProductCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    sku: str = Field(min_length=3, max_length=30)

    @field_validator("name")
    @classmethod
    def trim_name(cls, value: str) -> str:
        cleaned = value.strip()
        if not cleaned:
            raise ValueError("Name cannot contain only spaces")
        return cleaned
~~~

### Example 2: python

~~~python
from pydantic import BaseModel, Field, model_validator


class SaleCreate(BaseModel):
    price: float = Field(gt=0)
    discount_percent: int = Field(ge=0, le=100)

    @model_validator(mode="after")
    def reject_full_discount(self):
        if self.discount_percent == 100:
            raise ValueError("A full discount is not allowed")
        return self
~~~

### Example 3: python

~~~python
from pydantic import BaseModel, ConfigDict, Field


class PaymentCreate(BaseModel):
    model_config = ConfigDict(strict=True)

    amount_cents: int = Field(gt=0)
    currency: str = Field(min_length=3, max_length=3)
~~~

### Example 4: python

~~~python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

allowed_origins = [
    "http://localhost:5173",
    "https://app.example.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=allowed_origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    expose_headers=["X-Request-ID"],
    max_age=600,
)
~~~

### Example 5: python

~~~python
from fastapi import FastAPI
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["api.example.com", "localhost", "127.0.0.1"],
)
~~~

### Example 6: python

~~~python
from fastapi import FastAPI
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

app = FastAPI()
app.add_middleware(HTTPSRedirectMiddleware)
~~~


## 12. Authentication and authorization
[Open the chapter](./12-authentication-and-authorization.md)

### Example 1: sh

~~~sh
python -m pip install pyjwt
~~~

### Example 2: sh

~~~sh
python -m pip install "pwdlib[argon2]"
~~~

### Example 3: python

~~~python
from typing import Annotated

from fastapi import FastAPI, Security
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer

app = FastAPI()
bearer_scheme = HTTPBearer()


@app.get("/token-preview")
async def token_preview(
    credentials: Annotated[HTTPAuthorizationCredentials, Security(bearer_scheme)],
):
    return {"scheme": credentials.scheme}
~~~

### Example 4: python

~~~python
import os
from datetime import datetime, timedelta, timezone

import jwt
from jwt.exceptions import InvalidTokenError

JWT_SECRET = os.environ["JWT_SECRET"]
JWT_ISSUER = "https://identity.example.com/"
JWT_AUDIENCE = "books-api"


def create_access_token(subject: str) -> str:
    now = datetime.now(timezone.utc)
    claims = {
        "sub": subject,
        "iss": JWT_ISSUER,
        "aud": JWT_AUDIENCE,
        "iat": now,
        "exp": now + timedelta(minutes=15),
    }
    return jwt.encode(claims, JWT_SECRET, algorithm="HS256")


def verify_access_token(token: str) -> dict:
    return jwt.decode(
        token,
        JWT_SECRET,
        algorithms=["HS256"],
        issuer=JWT_ISSUER,
        audience=JWT_AUDIENCE,
        options={"require": ["sub", "exp", "iss", "aud"]},
    )
~~~

### Example 5: python

~~~python
from typing import Annotated

from fastapi import Depends, HTTPException, Security, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from jwt.exceptions import InvalidTokenError

bearer_scheme = HTTPBearer(auto_error=False)


async def get_current_user(
    credentials: Annotated[HTTPAuthorizationCredentials | None, Security(bearer_scheme)],
):
    if credentials is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Bearer token required",
            headers={"WWW-Authenticate": "Bearer"},
        )

    try:
        claims = verify_access_token(credentials.credentials)
    except InvalidTokenError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid access token",
            headers={"WWW-Authenticate": "Bearer"},
        )

    user = find_active_user(claims["sub"])
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Account is unavailable",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return user


CurrentUser = Annotated[dict, Depends(get_current_user)]


@app.get("/profile")
async def read_profile(user: CurrentUser):
    return {"id": user["id"], "name": user["name"]}
~~~

### Example 6: python

~~~python
from fastapi import HTTPException


def require_book_owner(book, user):
    if book.owner_id != user["id"]:
        raise HTTPException(
            status_code=403,
            detail="You cannot access this book",
        )
    return book
~~~

### Example 7: python

~~~python
from pwdlib import PasswordHash

password_hasher = PasswordHash.recommended()


def hash_password(password: str) -> str:
    return password_hasher.hash(password)


def verify_password(password: str, stored_hash: str) -> bool:
    return password_hasher.verify(password, stored_hash)
~~~


## 13. Background tasks and application lifespan
[Open the chapter](./13-background-tasks-and-lifespan.md)

### Example 1: python

~~~python
from fastapi import BackgroundTasks, FastAPI, status

app = FastAPI()


def write_notification(message: str):
    with open("notifications.log", "a", encoding="utf-8") as log_file:
        log_file.write(message + "\n")


@app.post("/reports/{report_id}/notify", status_code=status.HTTP_202_ACCEPTED)
async def notify_report_ready(
    report_id: int,
    background_tasks: BackgroundTasks,
):
    background_tasks.add_task(
        write_notification,
        f"Report {report_id} is ready",
    )
    return {"accepted": True, "report_id": report_id}
~~~

### Example 2: python

~~~python
from fastapi import BackgroundTasks, Depends, FastAPI

app = FastAPI()


def write_log_line(message: str):
    with open("request-events.log", "a", encoding="utf-8") as log_file:
        log_file.write(message + "\n")


def record_search(
    background_tasks: BackgroundTasks,
    q: str | None = None,
):
    if q:
        background_tasks.add_task(write_log_line, f"search={q}")
    return q


@app.get("/search")
async def search(q: str | None = Depends(record_search)):
    return {"query": q}
~~~

### Example 3: python

~~~python
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException, Request


@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.catalog = {
        1: {"id": 1, "name": "Notebook"},
    }
    try:
        yield
    finally:
        app.state.catalog.clear()


app = FastAPI(lifespan=lifespan)


@app.get("/catalog/{item_id}")
async def read_catalog_item(item_id: int, request: Request):
    item = request.app.state.catalog.get(item_id)
    if item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    return item
~~~


## 14. Testing FastAPI applications
[Open the chapter](./14-testing-fastapi-applications.md)

### Example 1: sh

~~~sh
python -m pip install pytest httpx
~~~

### Example 2: python

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

### Example 3: python

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

### Example 4: sh

~~~sh
python -m pytest
~~~

### Example 5: python

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

### Example 6: python

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

### Example 7: python

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

### Example 8: python

~~~python
def test_openapi_contains_books_route():
    client = TestClient(create_app())
    schema = client.get("/openapi.json").json()

    assert "/books" in schema["paths"]
    assert "post" in schema["paths"]["/books"]
~~~


## 15. Configuration, logging, and debugging
[Open the chapter](./15-configuration-logging-and-debugging.md)

### Example 1: sh

~~~sh
python -m pip install pydantic-settings python-dotenv
~~~

### Example 2: python

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

### Example 3: dotenv

~~~dotenv
BOOKS_APP_NAME=Local Books API
BOOKS_DEBUG=true
BOOKS_DATABASE_URL=sqlite:///./notes.db
BOOKS_ALLOWED_ORIGINS=["http://localhost:5173"]
~~~

### Example 4: python

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

### Example 5: python

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

### Example 6: sh

~~~sh
python -m uvicorn app.main:app --reload --log-level debug
~~~

### Example 7: sh

~~~sh
curl.exe -i "http://127.0.0.1:8000/books?page=1"
curl.exe -v http://127.0.0.1:8000/health
~~~


## 16. Deployment, performance, and operations
[Open the chapter](./16-deployment-performance-and-operations.md)

### Example 1: sh

~~~sh
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
~~~

### Example 2: sh

~~~sh
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2
~~~

### Example 3: sh

~~~sh
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --proxy-headers --forwarded-allow-ips=10.0.0.10
~~~

### Example 4: python

~~~python
from fastapi import FastAPI, HTTPException
from sqlalchemy import text
from sqlalchemy.exc import SQLAlchemyError

app = FastAPI()


@app.get("/health/live")
async def liveness():
    return {"status": "alive"}


@app.get("/health/ready")
def readiness():
    try:
        with SessionLocal() as session:
            session.execute(text("SELECT 1"))
    except SQLAlchemyError as exc:
        raise HTTPException(status_code=503, detail="Service is not ready") from exc
    return {"status": "ready"}
~~~


## Main references

- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [Python documentation](https://docs.python.org/3/)
- [Pydantic documentation](https://docs.pydantic.dev/latest/)
- [SQLAlchemy documentation](https://docs.sqlalchemy.org/en/20/)
