# 01. FastAPI and the ASGI request lifecycle

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Notes index](../README.md) | [Notes index](../README.md) | [Next: Python environment and the first API](./02-python-environment-and-first-api.md) |

## What FastAPI does

FastAPI is a Python framework for building HTTP APIs. You describe routes and the data they accept with Python functions and type annotations. FastAPI uses that information to validate requests, convert responses, and produce an OpenAPI description of the API.

FastAPI is built on Starlette for the web layer and Pydantic for data handling. It runs as an ASGI application, so it needs an ASGI server such as Uvicorn to accept network connections and call the app.

## Follow one request

A request passes through a few parts before the client receives a response:

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

The client might be a browser, mobile app, command-line tool, or another service. FastAPI handles the application-level request. Uvicorn handles the server connection and speaks the ASGI interface.

## Understand ASGI

ASGI means Asynchronous Server Gateway Interface. It is the standard connection between an asynchronous Python web server and an application.

The ASGI server calls an application with three values:

- `scope` describes the connection or request, such as its protocol, method, path, and headers.
- `receive` is used to receive request events, including body data.
- `send` is used to send response events back to the server.

For an HTTP request, the scope describes one request. For a WebSocket, the scope lasts for the connection. ASGI also defines a lifespan protocol that servers use to signal application startup and shutdown.

Most FastAPI code does not call `scope`, `receive`, or `send` directly. FastAPI and Starlette handle that protocol and expose a simpler request and response API.

## Create the first application

Save this as `main.py`:

~~~python
from fastapi import FastAPI

app = FastAPI(title="Reading List API")


@app.get("/")
async def read_home():
    return {"message": "Hello, FastAPI!"}
~~~

`FastAPI()` creates the application object. The `@app.get("/")` decorator registers a path operation for `GET /`. When a request matches, FastAPI calls `read_home()` and converts the returned dictionary into a JSON response.

## Start it with an ASGI server

Install FastAPI and Uvicorn in a Python environment, then start the server from the directory containing `main.py`:

~~~sh
uvicorn main:app --reload
~~~

The command `main:app` means: import the Python module named `main` and use the object named `app` from it. `--reload` watches files and restarts the server after changes. Use it for local development, not production.

Open these URLs while the server is running:

- `http://127.0.0.1:8000/` calls the route.
- `http://127.0.0.1:8000/docs` shows interactive API documentation.
- `http://127.0.0.1:8000/redoc` shows an alternative documentation view.
- `http://127.0.0.1:8000/openapi.json` returns the generated OpenAPI document.

FastAPI builds the OpenAPI description from route declarations, types, and later response and request models. The interactive documentation uses that description to show the API and let a developer try requests.

## Why the function can be asynchronous

Python's `async def` declares a coroutine function. Use `await` inside it when calling an asynchronous library:

~~~python
@app.get("/external-status")
async def read_external_status():
    result = await status_client.fetch()
    return {"status": result}
~~~

You can also use a regular `def` path operation when the library you call is synchronous and blocking:

~~~python
@app.get("/legacy-status")
def read_legacy_status():
    result = blocking_status_client.fetch()
    return {"status": result}
~~~

FastAPI runs normal `def` path operations and dependencies in a worker thread. This keeps their blocking I/O from blocking the event loop. A helper function that you call directly inside `async def` is called directly, so a blocking helper can still hold up that event loop. Choose `async def` when the operations you call support `await`; use `def` for blocking libraries unless you deliberately move that work to a worker.

Do not add `async` only because the framework supports it. The function body and the libraries it calls determine the correct choice. The database chapter explains the decision in more detail.

## What happens to a returned value

For ordinary values such as dictionaries, lists, strings, and Pydantic models, FastAPI turns the result into an HTTP response. A dictionary or list is serialized as JSON. Later chapters show how to declare a response model, choose a status code, and return errors.

Do not build JSON by joining strings. Return Python values and let FastAPI serialize them.

## FastAPI, Starlette, Pydantic, and Uvicorn

| Part | Responsibility |
| --- | --- |
| FastAPI | Route declarations, validation integration, dependency injection, and OpenAPI generation |
| Starlette | Web request and response features beneath FastAPI |
| Pydantic | Data parsing, validation, and serialization |
| Uvicorn | ASGI server that accepts connections and runs the application |

These parts have separate jobs. FastAPI is not a database, and Uvicorn is not the place to define application routes.

## Common questions at this stage

- **Do I call `app.run()`?** No. Start an ASGI server such as Uvicorn and point it at the application object.
- **Does `@app.get("/")` start the server?** No. It registers a route when Python imports the module.
- **Do I have to write a class for each route?** No. A path operation can be a regular function or an `async def` function.
- **Is the generated API documentation written by hand?** FastAPI generates it from the app's OpenAPI schema. You can add descriptions and examples as the API grows.
- **Does `async def` make blocking code asynchronous?** No. A blocking call inside an async function can still block its event loop.

## Main references

- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [FastAPI concurrency and async functions](https://fastapi.tiangolo.com/async/)
- [ASGI specification](https://asgi.readthedocs.io/en/latest/specs/main.html)
- [ASGI HTTP and WebSocket message format](https://asgi.readthedocs.io/en/latest/specs/www.html)
- [Python asyncio documentation](https://docs.python.org/3/library/asyncio.html)