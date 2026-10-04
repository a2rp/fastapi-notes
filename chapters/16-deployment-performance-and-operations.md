# 16. Deployment, performance, and operations

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Configuration, logging, and debugging](./15-configuration-logging-and-debugging.md) | [Notes index](../README.md) | [Next: All code samples](./98-all-code-samples.md) |

## Run the application for production

For local development, use reload mode so file changes restart the server. Production must use a stable process manager or hosting platform, fixed dependencies, protected configuration, and graceful restarts.

Start one Uvicorn process without reload:

~~~sh
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
~~~

`app.main:app` means the `app` object in `app/main.py`. Binding to `0.0.0.0` allows connections through the machine or container network. Restrict public access through the firewall, load balancer, or hosting platform.

## Use workers based on measurements

Uvicorn can start multiple worker processes:

~~~sh
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2
~~~

Each worker is a separate process with its own memory, application lifespan, connection pool, and in-memory state. More workers can use more CPU cores and handle more simultaneous requests, but they also use more memory and may duplicate startup work.

Choose worker counts from measurements and the resources available. If deployment runs one process per container, scale with the container platform instead of starting several workers inside every container unless that design has been tested.

## Put a TLS proxy in front of the app

A reverse proxy or load balancer commonly terminates HTTPS, renews certificates, and forwards requests to the ASGI server. The application may receive plain HTTP from that trusted internal proxy even while the client used HTTPS.

The proxy can send forwarded headers that describe the original client and scheme. Configure the ASGI server to trust these headers only from the proxy addresses you control:

~~~sh
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --proxy-headers --forwarded-allow-ips=10.0.0.10
~~~

Replace the example proxy address with the actual trusted address or private network. Trusting forwarded headers from every client allows a caller to forge the original scheme or client address.

Enable HTTPS at the public boundary, redirect insecure requests there, and use secure cookie settings if the app uses cookies. The proxy and app configuration must agree about the original request scheme to avoid redirect loops.

## Separate readiness from liveness

A liveness endpoint answers whether the process can respond. A readiness endpoint answers whether the instance should receive traffic, which may include a database or other required dependency check.

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

Use the real database session setup for the readiness check. Keep health responses free of credentials, internal hostnames, and detailed connection errors. Configure the platform to use the endpoint that matches its restart or traffic-routing decision.

## Run schema changes as a controlled step

Database migrations change persistent state. Run them as a deliberate deployment step before new application workers depend on the changed schema. Do not run migrations independently in every worker startup hook.

Use backward-compatible changes when old and new application versions may overlap during a rolling deploy. A common sequence is to add a nullable field, deploy code that writes both old and new forms, backfill records, switch reads, and remove the old field in a later release.

## Keep a deployment artifact reproducible

Record and install the application dependencies from a reviewed lock file or pinned dependency set. Build the same artifact for test and production where possible, and inject environment-specific settings at runtime.

Do not copy local `.env` files, test databases, access keys, development logs, or generated caches into a production image. Include only the code and runtime assets the service needs.

## Measure before tuning performance

Start with response time, error rate, request volume, CPU, memory, and database connection usage. Profile a slow endpoint before changing its implementation.

Common improvements include:

- Add database indexes for measured query patterns.
- Paginate large collections and select only required columns.
- Avoid repeated database calls inside loops.
- Reuse HTTP clients and connection pools through lifespan.
- Cache data only when its freshness and invalidation rules are clear.
- Move CPU-heavy or durable work to a separate worker system.

An `async def` endpoint improves concurrency only when its slow I/O calls are awaited and use asynchronous libraries. Blocking work inside an async route blocks that event loop. Use a regular `def` route for synchronous I/O or an appropriate worker for long-running computation.

## Plan for graceful shutdown and restarts

Use a process manager or hosting platform that restarts crashed workers and starts the application after machine restarts. Configure a graceful shutdown timeout long enough for requests to finish and resources to close.

Test deployment changes with health checks and logs before shifting all traffic. Keep rollback steps available for both application code and database changes.

## Protect the service at its boundaries

Set request size limits, timeouts, and rate limits at the application or edge proxy. Restrict administrative routes, use HTTPS, validate trusted hosts, and keep dependency versions updated.

Do not return stack traces or raw database errors to clients. Collect operational logs and metrics without recording passwords, tokens, or private request bodies.

## Deployment checklist

- Debug and reload modes are disabled.
- Required settings and secrets come from protected runtime configuration.
- Public traffic uses HTTPS.
- Forwarded headers are trusted only from known proxies.
- Readiness and liveness checks match the hosting platform.
- Database migrations run once as a deployment step.
- Worker count and memory use were measured under representative load.
- Logs contain useful request context without sensitive data.
- A rollback path exists for the app and its schema.

## Practice

1. Start the app without reload and verify that it listens on the expected interface and port.
2. Run two workers and observe how process-local state differs.
3. Configure forwarded headers for a known proxy address and test the original scheme.
4. Add a readiness check for a test database and verify it returns `503` when unavailable.
5. Record response time and error rate for a representative request before tuning it.

## Main references

- [FastAPI deployment concepts](https://fastapi.tiangolo.com/deployment/concepts/)
- [FastAPI HTTPS and proxy headers](https://fastapi.tiangolo.com/deployment/https/)
- [FastAPI server workers](https://fastapi.tiangolo.com/deployment/server-workers/)
- [Uvicorn deployment](https://www.uvicorn.org/deployment/)
- [Uvicorn settings](https://www.uvicorn.org/settings/)
