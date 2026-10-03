# 13. Background tasks and application lifespan

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Authentication and authorization](./12-authentication-and-authorization.md) | [Notes index](../README.md) | [Next: Testing FastAPI applications](./14-testing-fastapi-applications.md) |

## Run small work after a response

`BackgroundTasks` schedules a function to run after FastAPI has sent the response. It is useful when a small follow-up action should not delay the client, such as writing a notification record or sending a simple email.

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

FastAPI runs the task after the response is sent. The task can be a normal function or an async function. This example writes a local log for practice; a real notification should use a managed mail or messaging service.

## Pass only the data a task needs

Pass stable values such as an identifier, address, or small payload. Do not pass a request object, open request upload, or a database session that will be closed during request cleanup.

If a task needs a database connection, have the task open its own session. If the work can take a long time, the response can acknowledge the request while a separate worker processes a durable job.

## Understand what an in-process task guarantees

`BackgroundTasks` runs in the application process after the response. It is not a durable job queue. If the process stops, the server restarts, or the task raises an exception, the application does not automatically retry the work.

Use a separate queue and worker system for important or heavy work that needs retries, scheduling, progress tracking, or execution across multiple machines. Store a job record before returning `202 Accepted` so the client can check its state.

## Combine tasks added by dependencies

A dependency can add to the same `BackgroundTasks` object used by the endpoint:

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

The dependency and endpoint can add tasks to the shared object. Do not use this to build a hidden job system; use an explicit queue when the work needs durable execution or reliable retries.

## Initialize shared resources with lifespan

Application lifespan is an async context manager. Code before `yield` runs before the application accepts requests. Code after `yield` runs during shutdown:

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

Use lifespan for resources shared by requests, such as a connection pool, HTTP client, or in-memory read-only data. Pair every acquired resource with cleanup in a `finally` block.

For HTTP clients and database engines, close or dispose of the resource after `yield`. Do not create a new shared connection pool inside every route.

## Keep lifespan work safe with multiple workers

Each application worker is a separate process. Its lifespan code runs once in that process, so an in-memory cache is not shared between workers. A periodic scheduler started in lifespan could also run once per worker and duplicate jobs.

Use a shared external store for state that must be visible across workers. Run singleton schedules in a dedicated process or use a scheduler designed for multiple workers.

## Use lifespan when resources need a clear lifetime

Startup and shutdown event decorators exist in older applications. For new resource setup, use the `lifespan` parameter so setup and cleanup live together in one context. Do not configure both lifespan and separate event handlers for the same resource lifecycle.

FastAPI test clients can trigger startup and shutdown when used as context managers. The next chapter shows how to test behavior that depends on initialized resources.

## Practice

1. Add a small background notification task and confirm the endpoint responds with its acknowledgement.
2. Make the task open its own resource instead of reusing a request-scoped database session.
3. Identify whether the task may be lost if the server stops and decide if a durable queue is required.
4. Add a lifespan resource and cleanup in a `finally` block.
5. Run the app with more than one worker and identify which data is process-local.

## Main references

- [FastAPI background task reference](https://fastapi.tiangolo.com/reference/background/)
- [FastAPI lifespan events](https://fastapi.tiangolo.com/advanced/events/)
- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [Starlette background tasks](https://www.starlette.io/background/)
- [Python asynchronous context managers](https://docs.python.org/3/library/contextlib.html)
