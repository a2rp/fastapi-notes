# FastAPI Study Notes

These are my personal study notes from learning and working with FastAPI. I am collecting Python concepts, API patterns, and practical examples that help me understand and build HTTP services.

## About this collection

This repository is a working record of what I study and practice with FastAPI. The notes explain the request lifecycle, core framework features, common integrations, and the decisions that help an API remain clear and maintainable.

The examples use Python and FastAPI. Each chapter builds on the previous topics, with runnable examples and plain explanations. The goal is to keep useful notes in one place for later study and project work.

## Core topics

- Python environments and ASGI
- Routing and request data
- Pydantic models and validation
- Dependencies and response handling
- Errors, forms, uploads, and static files
- Databases and asynchronous work
- Security, authentication, and authorization
- Testing, configuration, and deployment

## Chapters

01. [FastAPI and the ASGI request lifecycle](./chapters/01-fastapi-and-asgi.md)  
   Understand FastAPI, ASGI, the server, and how a request reaches an endpoint.

02. [Python environment and the first API](./chapters/02-python-environment-and-first-api.md)  
   Create a Python environment, install FastAPI and Uvicorn, and start a small API.

03. [Path operation routing](./chapters/03-path-operation-routing.md)  
   Define path operations, HTTP methods, route order, and modular routers.

04. [Path, query, and header parameters](./chapters/04-path-query-and-header-parameters.md)  
   Declare and validate values from paths, query strings, and request headers.

05. [Request bodies and Pydantic models](./chapters/05-request-bodies-and-pydantic-models.md)  
   Parse JSON request bodies with Pydantic models and field constraints.

06. [Dependencies and reusable logic](./chapters/06-dependencies-and-reusable-logic.md)  
   Use dependency injection for shared services, authentication, and request-scoped resources.

07. [Response models and status codes](./chapters/07-response-models-and-status-codes.md)  
   Control response shapes, serialization, headers, and HTTP status codes.

08. [Errors and exception handling](./chapters/08-errors-and-exception-handling.md)  
   Return clear HTTP errors and handle application exceptions consistently.

09. [Forms, files, and static resources](./chapters/09-forms-files-and-static-resources.md)  
   Receive form data and uploads, and serve static files safely.

10. [Database integration and async I/O](./chapters/10-database-integration-and-async-io.md)  
   Connect endpoints to a database and choose synchronous or asynchronous I/O.

11. [Validation, security, and CORS](./chapters/11-validation-security-and-cors.md)  
   Validate untrusted input and apply practical API security controls.

12. [Authentication and authorization](./chapters/12-authentication-and-authorization.md)  
   Establish user identity, protect routes, and enforce record permissions.

13. [Background tasks and application lifespan](./chapters/13-background-tasks-and-lifespan.md)  
   Manage startup and shutdown resources and schedule work outside a response.

14. [Testing FastAPI applications](./chapters/14-testing-fastapi-applications.md)  
   Test path operations, dependencies, validation, and database behavior.

15. [Configuration, logging, and debugging](./chapters/15-configuration-logging-and-debugging.md)  
   Validate settings, structure logs, and investigate request failures.

16. [Deployment, performance, and operations](./chapters/16-deployment-performance-and-operations.md)  
   Run FastAPI in production with a process manager, health checks, and safe shutdown.

## Reference chapters

- [All code samples](./chapters/98-all-code-samples.md) collects examples from the core chapters.
- [Complete questions and answers](./chapters/99-complete-q-and-a.md) gathers review questions across the notes.

## How to use these notes

Read the chapters in order when learning FastAPI, or open the section that answers a question from your current project. Run each example in a small local environment, change one value at a time, and compare the response with the explanation.

## Main references

- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [Python documentation](https://docs.python.org/3/)
- [Pydantic documentation](https://docs.pydantic.dev/latest/)
- [Uvicorn documentation](https://www.uvicorn.org/)

## License

These notes are available under the [MIT License](./LICENSE).

## Links

- Portfolio: https://www.ashishranjan.net
- GitHub: https://github.com/a2rp
- CodePen: https://codepen.io/ash1198
- LinkedIn: https://www.linkedin.com/in/aashishranjan
- Facebook: https://www.facebook.com/theash.ashish/
- YouTube: https://www.youtube.com/@ashishranjan-ashz?sub_confirmation=1
- Email: mailto:ash.ranjan09@gmail.com

## Support

- Support: https://a2rp-donation-page.netlify.app/
- Buy Me a Coffee: https://buymeacoffee.com/ashishranjan
- Patreon: https://www.patreon.com/ashishranjan
