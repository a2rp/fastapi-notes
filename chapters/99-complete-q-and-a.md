# 99. Complete questions and answers

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: All code samples](./98-all-code-samples.md) | [Notes index](../README.md) | End of notes |

These questions review the core ideas from all 16 chapters. Answers focus on the reasoning behind common FastAPI decisions, not only syntax.

## 01. FastAPI and the ASGI request lifecycle

### Q1. What is FastAPI?
FastAPI is a Python framework for building HTTP APIs. It uses Python type annotations to validate inputs, document operations, and describe responses.

### Q2. What is ASGI?
ASGI is the asynchronous server interface between a Python web application and an application server. It supports HTTP and WebSocket communication.

### Q3. What does Uvicorn do?
Uvicorn is an ASGI server. It accepts network connections and passes requests to an ASGI application such as a FastAPI app.

### Q4. What role does Starlette have?
FastAPI builds on Starlette for core web behavior such as routing, middleware, request handling, and response support.

### Q5. What happens when a request reaches a path operation?
The server passes the request to the ASGI app, middleware runs, FastAPI matches the route, resolves inputs and dependencies, calls the endpoint, then creates a response.

### Q6. Does `async def` make every operation faster?
No. It helps when the endpoint awaits non-blocking I/O. Blocking database calls or CPU-heavy work still occupy execution resources.

### Q7. How are `/docs` and `/openapi.json` related?
FastAPI generates an OpenAPI schema at `/openapi.json`. The interactive `/docs` page reads that schema to display the API contract.

## 02. Python environment and the first API

### Q1. Why use a virtual environment?
A virtual environment keeps one project Python packages separate from other projects and from the system Python installation.

### Q2. How do you create and activate one on Windows?
Run `py -m venv .venv`, then activate it in PowerShell with `.\.venv\Scripts\Activate.ps1`.

### Q3. How do you install FastAPI and Uvicorn?
Install them into the active environment with `python -m pip install fastapi uvicorn`.

### Q4. What does `app = FastAPI()` create?
It creates the ASGI application object that the server imports and runs.

### Q5. What does `uvicorn app:app --reload` mean?
The first `app` is the Python module. The second `app` is the application object inside that module. Reload watches code changes for development.

### Q6. Why should reload stay off in production?
Reload adds a development file watcher and restarts the process on changes. Production should use a managed process configured for stable operation.

### Q7. Why record project dependencies?
A reviewed dependency list or lock file lets another environment install the packages needed to run the application.

## 03. Path operation routing

### Q1. What is a path operation?
It connects an HTTP method and URL path to a Python function, such as `GET /books` to a function that lists books.

### Q2. What is the difference between a path and a method?
The path identifies the resource location. The method describes the kind of operation, such as reading with `GET` or creating with `POST`.

### Q3. Why register fixed paths before parameterized paths?
Routes are checked in order. A general parameter path can match text intended for a more specific route if the specific route comes later.

### Q4. What is `APIRouter` used for?
It groups related operations so a larger app can organize routes by feature and include them under prefixes, tags, or shared dependencies.

### Q5. Does the Python function name define the URL?
No. The decorator path defines the URL. A clear function name helps people understand the operation in source code.

### Q6. When can two operations share a URL path?
They can share a path when they use different HTTP methods, such as `GET /books` and `POST /books`.

### Q7. What is a route prefix?
A prefix is a path segment applied when including a router. For example, a router with `/books` included under `/api/v1` serves `/api/v1/books`.

## 04. Path, query, and header parameters

### Q1. Where does a path parameter come from?
It comes from a named placeholder in the route, such as `{book_id}` in `/books/{book_id}`.

### Q2. Where does a query parameter come from?
It comes after `?` in the URL, such as `page=2` in `/books?page=2`.

### Q3. How is an optional query parameter declared?
Give it a default, often `None`. A parameter without a default is required.

### Q4. What does `Query(ge=1, le=100)` do?
It requires a numeric value between 1 and 100, inclusive, and adds those rules to the API schema.

### Q5. How can a query parameter accept repeated values?
Annotate it as a list and declare it with `Query()`. A URL can then repeat the key, such as `tag=python&tag=api`.

### Q6. How does FastAPI read a header parameter?
Declare it with `Header()`. Header names are case-insensitive, and Python underscores map to hyphens by default.

### Q7. What happens when a parameter cannot be converted or validated?
FastAPI rejects the request before calling the endpoint and returns a request validation response, normally with status `422`.

## 05. Request bodies and Pydantic models

### Q1. What does a Pydantic request model describe?
It describes the accepted shape and validation rules for structured request data such as a JSON object.

### Q2. Which model fields are required?
A field without a default is required. Giving a field a default allows the client to omit it.

### Q3. Does `str | None` always make a field optional?
No. It allows `null`. Without a default the field is still required. Use `str | None = None` to allow omission as well.

### Q4. Why use `Field(default_factory=list)`?
It creates a fresh list for each model instance instead of sharing one mutable default.

### Q5. What is a nested model?
It is a Pydantic model used as a field inside another model. It validates nested JSON objects and collections recursively.

### Q6. What does `Body(embed=True)` change?
It wraps one scalar body value under its parameter name, such as `{"enabled": true}` instead of a top-level JSON boolean.

### Q7. What does `model_dump(exclude_unset=True)` preserve?
It includes only fields that the caller sent, which lets a partial update distinguish an omitted field from an explicit `null`.

## 06. Dependencies and reusable logic

### Q1. What does `Depends()` do?
It tells FastAPI to call a dependency and inject its result into a route or another dependency.

### Q2. Why use a dependency?
It makes shared work such as pagination, authentication, settings, and database sessions reusable and testable.

### Q3. Why use `Annotated` with a dependency?
It keeps the Python type and FastAPI dependency declaration together while preserving a useful type for editors and type checkers.

### Q4. Can a class be a dependency?
Yes. FastAPI calls the class with the resolved parameters, which can be useful when related values belong in one object.

### Q5. What is a sub-dependency?
It is a dependency requested by another dependency. FastAPI resolves the dependency graph before calling the endpoint.

### Q6. How does dependency caching work?
FastAPI normally reuses one dependency result during a request when that dependency is needed more than once. `use_cache=False` requests another call.

### Q7. Why use `yield` in a dependency?
It provides a value to the request and runs cleanup afterward, which is useful for closing sessions or other request-scoped resources.

## 07. Response models and status codes

### Q1. What does `response_model` do?
It documents, validates, serializes, and filters the endpoint output to a declared public shape.

### Q2. Why separate input and output models?
The database or input may contain private fields that should not appear in a public response.

### Q3. What happens if the returned data does not match the response model?
It indicates an application error. FastAPI reports a server-side failure rather than treating it as invalid client input.

### Q4. Which status is common for a newly created resource?
`201 Created` indicates that the request created a resource.

### Q5. What does `204 No Content` mean?
The operation succeeded and the response has no body.

### Q6. Does returning `JSONResponse` use response model filtering automatically?
No. Returning a response object directly bypasses normal response model conversion and filtering.

### Q7. How can a route map an ORM object to a Pydantic response?
With Pydantic v2, configure `from_attributes=True` and validate the object with `model_validate()` or return it through a matching response model.

## 08. Errors and exception handling

### Q1. How does an endpoint return a known HTTP error?
Raise `HTTPException` with a status code and safe detail. The exception stops the current request flow.

### Q2. When is `404` appropriate?
Use it when a requested resource does not exist, or when the API intentionally hides whether a protected resource exists.

### Q3. What is the difference between `401` and `403`?
`401` means valid authentication is required or missing. `403` means the caller is known but lacks permission.

### Q4. When should a custom exception handler be used?
Use one when a domain error should map consistently to an HTTP response across multiple routes.

### Q5. What is `RequestValidationError`?
It is the exception FastAPI raises for invalid request input such as a bad path value or body field.

### Q6. Why avoid returning raw request bodies in validation errors?
Bodies may contain passwords, tokens, or personal information that should not be repeated to a client or copied into logs.

### Q7. Should unexpected exceptions be converted into `200 OK`?
No. Keep unexpected failures as server errors, log useful context securely, and avoid exposing internal tracebacks.

## 09. Forms, files, and static resources

### Q1. Which package is needed to parse form and file uploads?
FastAPI relies on `python-multipart` for multipart form parsing.

### Q2. Why cannot one request body be both JSON and multipart form data?
An HTTP request body has one encoding. Use form fields for multipart data or send JSON in a separate request.

### Q3. When should a file be typed as `bytes`?
Use `bytes` only for predictably small files because FastAPI reads the whole file into memory.

### Q4. Why use `UploadFile`?
It exposes file metadata and a spooled file interface that can handle larger contents without loading the whole file into memory at once.

### Q5. Is the uploaded filename or content type trustworthy?
No. Both are supplied by the client. Generate a server-side name and inspect file content with a suitable parser.

### Q6. What is `StaticFiles` for?
It mounts a directory of files that are intentionally public, such as CSS or public images.

### Q7. How should an API return a private report file?
Use a server-controlled path, check that the caller is authorized, and restrict requested names with an allowlist or trusted identifier.

## 10. Database integration and async I/O

### Q1. What is the difference between an engine and a session?
An engine manages database connections and pooling. A session tracks database operations and transaction state for a unit of work.

### Q2. Why create a session per request?
A session is mutable and stateful. A request-scoped session avoids unsafe sharing across requests or concurrent tasks.

### Q3. What does `commit()` do?
It commits pending changes in the current transaction to the database.

### Q4. Why call `refresh()` after inserting a record?
It reloads values generated by the database, such as a primary key.

### Q5. What must happen after an `IntegrityError`?
Roll back the session transaction before using that session again, then map only known conflicts to safe API errors.

### Q6. Does `Base.metadata.create_all()` replace migrations?
No. It is useful for local learning but does not manage schema evolution. Use migrations for existing application databases.

### Q7. Can one `AsyncSession` be shared by concurrent tasks?
No. Each concurrent task should use its own session because an `AsyncSession` is mutable and represents one transaction.

## 11. Validation, security, and CORS

### Q1. What is the purpose of request validation?
It checks the accepted types, shape, and constraints before the application uses client-supplied values.

### Q2. When should a custom validator be used?
Use it for a small deterministic rule that cannot be expressed with built-in type and field constraints.

### Q3. What does CORS control?
CORS tells browsers which origins may read cross-origin responses. It does not authenticate callers or replace authorization.

### Q4. What makes two URLs different origins?
An origin includes the scheme, host, and port. Changing any one of them creates a different origin.

### Q5. What is a preflight request?
It is a browser `OPTIONS` request that asks whether a cross-origin method and its headers are allowed.

### Q6. Why use explicit origins when credentials are enabled?
Credentialed browser requests need a specific allowed origin. A wildcard does not safely identify which origin may use credentials.

### Q7. What do trusted-host and HTTPS redirect middleware protect?
Trusted-host middleware restricts accepted `Host` values. HTTPS redirect middleware sends insecure requests to a secure scheme when proxy setup supports it.

## 12. Authentication and authorization

### Q1. What is the difference between authentication and authorization?
Authentication verifies who the caller is. Authorization checks what that caller may do.

### Q2. Does `HTTPBearer` verify a token?
No. It extracts the bearer value and documents the scheme. The application must verify the token signature and claims.

### Q3. Is a signed JWT encrypted?
Usually no. Its claims can be read by anyone who has the token. A signature detects changes but does not hide content.

### Q4. Which token claims should an API commonly validate?
Validate the signature and expected algorithm, expiry, issuer, audience, and required subject before trusting the token.

### Q5. Why use an identity provider with Authorization Code and PKCE?
It keeps sign-in and token issuance in a purpose-built provider, while PKCE protects the authorization code flow for public clients.

### Q6. How should a service store user passwords?
Use a salted, adaptive password hashing algorithm such as Argon2id through a maintained password hashing library. Never store plain passwords.

### Q7. Does a valid token prove ownership of a record?
No. The API must check ownership or permissions for each protected operation using trusted server-side identity data.

## 13. Background tasks and application lifespan

### Q1. When are FastAPI background tasks run?
They run in the application process after the response is sent.

### Q2. Are `BackgroundTasks` a durable queue?
No. Work can be lost if the process stops, and the framework does not provide durable retries or cross-machine execution.

### Q3. What values should be passed to a background task?
Pass the small values the task needs, such as an identifier or message. Do not pass a request-scoped session or open request file.

### Q4. What happens before and after `yield` in lifespan?
The code before `yield` initializes shared resources before requests are accepted. Code after it cleans them up during shutdown.

### Q5. What belongs in application lifespan?
Resources shared across requests, such as a connection pool, reusable client, or in-memory read-only data.

### Q6. What changes when the app uses several worker processes?
Each worker has separate memory and runs its own lifespan. A scheduler started there may run once per worker.

### Q7. How can a test run lifespan startup and cleanup?
Use `TestClient` as a context manager or use a compatible lifespan manager with an asynchronous ASGI client.

## 14. Testing FastAPI applications

### Q1. What is `TestClient`?
It is an in-process client for sending HTTP requests to a FastAPI app without opening a network port.

### Q2. What should an API test assert?
Assert observable behavior such as status code, response fields, headers, and validation outcome.

### Q3. Why create a new app for tests?
A fresh app gives each test isolated state and reduces order-dependent failures.

### Q4. How can a test replace a dependency?
Set `app.dependency_overrides[original]` to a fake dependency, then restore the original mapping after the test.

### Q5. Why use `TestClient` as a context manager?
It runs application startup and shutdown, which is required for tests using lifespan-managed resources.

### Q6. When is an async HTTPX client useful?
Use it when the test itself must await asynchronous functions or resources in addition to making API requests.

### Q7. Why should tests avoid checking the full validation message?
Error wording can change across library versions. Check stable fields such as status, location, and error type instead.

## 15. Configuration, logging, and debugging

### Q1. Why read settings from environment variables?
They allow development, test, and production to use different configuration without editing application source.

### Q2. What does `env_prefix="BOOKS_"` do?
It maps the settings field `database_url` to the environment variable `BOOKS_DATABASE_URL`.

### Q3. Should a real `.env` file be committed?
No. Ignore it in Git and provide a placeholder `.env.example` without working secrets.

### Q4. Why use a settings dependency?
It centralizes configuration and lets tests replace settings without changing global application code.

### Q5. When should `logger.exception()` be called?
Call it inside an exception handler when a traceback is useful for server-side diagnosis.

### Q6. Which values must stay out of logs?
Do not log passwords, access tokens, cookies, credentials, payment details, or full private request bodies.

### Q7. Why keep debug mode off in production?
Debug responses can expose implementation details and tracebacks. Send details to protected logs instead.

## 16. Deployment, performance, and operations

### Q1. Why should production run without reload?
Reload is a development file watcher that consumes resources and is not intended to supervise a production service.

### Q2. What is a Uvicorn worker?
A worker is a separate process running the application. Workers use separate memory and resources.

### Q3. Should worker count always match CPU count?
No. Measure throughput and memory under representative load, then choose a count that fits the deployment.

### Q4. Why trust proxy headers only from known addresses?
Forwarded headers can be forged by clients. Trusting only the controlled proxy preserves the real scheme and client address.

### Q5. What is the difference between liveness and readiness?
Liveness indicates that the process can respond. Readiness indicates whether the instance should receive traffic, including required dependency checks.

### Q6. Why run migrations separately from worker startup?
Several workers may start at once. Running schema changes in each process can race or repeat destructive operations.

### Q7. When does asynchronous I/O improve an endpoint?
It can improve concurrency when the endpoint awaits non-blocking I/O. It does not speed up CPU-heavy work or blocking calls.

## Main references

- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [Python documentation](https://docs.python.org/3/)
- [Pydantic documentation](https://docs.pydantic.dev/latest/)
- [SQLAlchemy documentation](https://docs.sqlalchemy.org/en/20/)
- [Uvicorn documentation](https://www.uvicorn.org/)
