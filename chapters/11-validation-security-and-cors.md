# 11. Validation, security, and CORS

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Database integration and async I/O](./10-database-integration-and-async-io.md) | [Notes index](../README.md) | [Next: Authentication and authorization](./12-authentication-and-authorization.md) |

## Validate data at the API boundary

Every value from a URL, header, form, or request body is client-controlled. Use type annotations and model constraints to reject malformed values before application logic uses them.

Field constraints handle common rules such as length and numeric ranges. A custom validator is useful when a value needs a small normalization step or a rule that cannot be expressed as one field constraint.

## Normalize and check one field

Pydantic v2 uses `field_validator` for custom field rules:

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

Keep validators small and deterministic. They should not make database calls or depend on mutable application state. Put those checks in a service or dependency where the failure can be handled with the correct HTTP response.

## Validate a rule across multiple fields

Use a model validator for a rule that depends on more than one field:

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

Use the narrowest rule that matches the business requirement. A validation error from the model becomes a request validation response. A rule based on database state, permissions, or another changing value belongs in application logic.

## Decide whether coercion is acceptable

Pydantic converts some compatible inputs by default. For example, a numeric string may become an integer. This can make APIs easier to use, but a strict contract may need to reject values of the wrong input type.

Strict mode can be applied to one field or configured for a model:

~~~python
from pydantic import BaseModel, ConfigDict, Field


class PaymentCreate(BaseModel):
    model_config = ConfigDict(strict=True)

    amount_cents: int = Field(gt=0)
    currency: str = Field(min_length=3, max_length=3)
~~~

Choose strictness deliberately and test actual JSON requests. Query and path values arrive as text, so strict handling for URL parameters can differ from strict handling for JSON bodies.

## Keep validation separate from authorization

Validation answers whether a value has an accepted shape. Authorization answers whether the current caller may perform the requested action on a particular resource. A valid account identifier does not prove that the caller owns that account.

Check ownership and permissions on the server after authentication. Never use a client-supplied user ID or role as proof of identity.

## Understand browser origins

An origin is the combination of scheme, host, and port. A page served from `https://app.example.com` and an API at `https://api.example.com` have different origins, even though both use HTTPS.

Browsers apply Cross-Origin Resource Sharing rules to frontend JavaScript requests. CORS tells a browser which origins may read a response. It does not authenticate a user, block non-browser clients, or replace server authorization.

## Allow only the frontend origins you use

Add `CORSMiddleware` with explicit origins, methods, and headers:

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

List the exact schemes, hosts, and ports that should be able to call the API from a browser. A development origin such as `http://localhost:5173` is different from `http://localhost:3000`.

When credentials are enabled, configure explicit origins, methods, and headers instead of using a wildcard. Cookies also need appropriate browser cookie settings, and cross-site cookie use requires careful CSRF protection.

## Understand preflight requests

A browser may first send an `OPTIONS` preflight request to ask whether a cross-origin method and its headers are allowed. `CORSMiddleware` handles this request and returns the allowed origin, methods, and headers.

If a browser request fails, inspect the browser network panel and response headers. Confirm the exact `Origin`, requested method, requested headers, and the API middleware settings. Adding `*` without understanding the request can hide the configuration problem and widen access unnecessarily.

## Validate the `Host` header

`TrustedHostMiddleware` rejects requests whose `Host` header is not in the allowlist:

~~~python
from fastapi import FastAPI
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["api.example.com", "localhost", "127.0.0.1"],
)
~~~

Use the actual public hostnames and local development hosts. Do not allow every host in a production deployment unless that behavior is intentional.

## Redirect HTTP requests to HTTPS

`HTTPSRedirectMiddleware` redirects HTTP and WebSocket connections to their secure schemes:

~~~python
from fastapi import FastAPI
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

app = FastAPI()
app.add_middleware(HTTPSRedirectMiddleware)
~~~

Enable this when the application can correctly determine the original scheme. Behind a reverse proxy, configure trusted proxy headers only for the proxy you control. Incorrect proxy configuration can create redirect loops or trust spoofed scheme headers.

In production, TLS is often terminated at a load balancer or proxy. Configure HTTPS at that boundary as well as the application behavior that matches the deployment.

## Use layers that match the threat

- Pydantic types and constraints check request shape and values.
- CORS controls which browser origins can read cross-origin responses.
- Authentication identifies the caller.
- Authorization checks whether that caller may perform an action.
- HTTPS protects traffic in transit.
- Trusted host settings constrain accepted host headers.

No single layer replaces the others. The next chapter focuses on authentication and authorization flows.

## Practice

1. Add a validator that trims a display name and rejects an empty result.
2. Add a model-level rule that compares two fields.
3. Test a numeric JSON field with both a number and a numeric string in strict mode.
4. Configure CORS for one local frontend and one deployed frontend origin.
5. Send a browser preflight request and inspect the returned headers.
6. Add an allowed host and reject a request using an unknown host.

## Main references

- [FastAPI middleware reference](https://fastapi.tiangolo.com/reference/middleware/)
- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [Pydantic validators](https://docs.pydantic.dev/latest/concepts/validators/)
- [Pydantic strict mode](https://docs.pydantic.dev/latest/concepts/strict_mode/)
- [Starlette middleware](https://www.starlette.io/middleware/)
