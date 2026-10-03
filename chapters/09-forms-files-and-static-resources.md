# 09. Forms, files, and static resources

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Errors and exception handling](./08-errors-and-exception-handling.md) | [Notes index](../README.md) | [Next: Database integration and async I/O](./10-database-integration-and-async-io.md) |

## Form data is different from JSON

HTML forms usually send `application/x-www-form-urlencoded`. A form that includes a file sends `multipart/form-data`. FastAPI uses `Form()` to read regular fields and `File()` or `UploadFile` to receive uploaded files.

Install the multipart parser in the project environment before adding form or file endpoints:

~~~sh
python -m pip install python-multipart
~~~

A request body has one encoding. An endpoint that reads multipart form data cannot also expect a separate JSON body in the same request. Put related values in form fields or use a separate JSON endpoint.

## Read form fields

Declare each submitted value with `Form()`:

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

Do not return the password or write it to logs. This example only shows how a form field reaches an endpoint; the authentication chapter covers checking credentials safely.

## Accept a small upload as bytes

Annotating a file as `bytes` is convenient for small uploads, but FastAPI reads the entire file into memory:

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

Use this only when the file size is predictably small. A large request can consume significant memory.

## Use `UploadFile` for file metadata and bounded reads

`UploadFile` provides the filename, content type, size when available, and an async file interface. Its spooled file keeps smaller uploads in memory and can move larger contents to disk:

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

The read limit prevents this example from loading an unbounded file into memory. For larger files, read and process chunks rather than calling `await file.read()` with no size. Finish processing an upload during the request, or copy it to controlled storage before scheduling work that continues later.

## Combine file and form fields

Use `File()` and `Form()` together when a form submits both a file and text:

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

Try it from a terminal with a local image file:

~~~sh
curl.exe -i http://127.0.0.1:8000/profile-photo -F "display_name=Sam" -F "file=@avatar.png"
~~~

## Treat every uploaded file as untrusted

A client controls the supplied filename and content type. Neither proves that the bytes are a safe image, document, or archive. A safer storage flow should:

- Enforce request and file size limits at the application and, where possible, the reverse proxy.
- Generate a server-side storage name rather than using the submitted filename as a path.
- Check allowed extensions and inspect file signatures with a suitable parser.
- Store uploads outside the source tree and outside any executable or public directory.
- Apply authorization and malware scanning where the application needs them.
- Avoid returning local filesystem paths or sensitive upload metadata.

Do not concatenate the client filename into a path. A filename may contain separators or other values intended to escape the upload directory.

## Serve intentional public files

Use `StaticFiles` for assets that are meant to be publicly readable, such as a CSS file or a public logo:

~~~python
from pathlib import Path

from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

app = FastAPI()
Path("static").mkdir(exist_ok=True)
app.mount("/static", StaticFiles(directory="static"), name="static")
~~~

With a file named `static/site.css`, the URL is `/static/site.css`. A mounted static application handles its path prefix separately from the main API routes and is not included as normal API operations in the OpenAPI schema.

Do not mount a directory containing private documents or user uploads unless every file in it is intended for anyone who can reach the server.

## Return a file as a response

For a generated or stored file response, use `FileResponse` and a server-controlled path:

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

The allowlist prevents a caller from choosing an arbitrary path. In a real system, authorize access to each report before returning it.

## Practice

1. Add a form endpoint with two validated text fields.
2. Receive a small file as `UploadFile` and return its measured byte length.
3. Enforce a size limit and reject a file that exceeds it.
4. Submit a file and a text field together with `curl.exe -F`.
5. Mount a directory containing only public assets, then request one asset URL.
6. Add an allowlist and an authorization check before returning a private report.

## Main references

- [FastAPI request parameter reference](https://fastapi.tiangolo.com/reference/parameters/)
- [FastAPI `UploadFile` reference](https://fastapi.tiangolo.com/reference/uploadfile/)
- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [Starlette static files](https://www.starlette.io/staticfiles/)
- [Python pathlib](https://docs.python.org/3/library/pathlib.html)
