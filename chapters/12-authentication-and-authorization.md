# 12. Authentication and authorization

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Validation, security, and CORS](./11-validation-security-and-cors.md) | [Notes index](../README.md) | [Next: Background tasks and application lifespan](./13-background-tasks-and-lifespan.md) |

## Authentication and authorization answer different questions

Authentication checks who is making a request. Authorization checks what that authenticated caller may do. A valid access token does not automatically grant access to every account or record.

A common API request sends an access token in the `Authorization: Bearer ...` header. The API must verify that token before trusting its claims.

## Prefer an established identity provider for sign-in

For a new application with browser or mobile clients, use an established OpenID Connect identity provider and the Authorization Code flow with PKCE. Let the provider handle sign-in, multi-factor options, recovery, and token issuance. The FastAPI service acts as a resource server that verifies access tokens.

The OAuth 2.0 Resource Owner Password Credentials flow sends the user password to the client and is not a good choice for a new system. The code below focuses on how a resource server reads and verifies bearer tokens.

## Install token verification tools

For the local signed-token example below, install PyJWT:

~~~sh
python -m pip install pyjwt
~~~

If your service stores passwords itself, install an adaptive password hashing library as well:

~~~sh
python -m pip install "pwdlib[argon2]"
~~~

Never store a plain password. Never use a fast general-purpose digest such as SHA-256 as a password hash. A password hashing library chooses a slow, salted algorithm intended for this job.

## Read the bearer token from a request

`HTTPBearer` extracts the bearer credential and describes the scheme in the generated API schema:

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

This only reads the header. It does not verify a signature, expiry, issuer, audience, or permission. Never authorize a request based only on the fact that a bearer string was present.

## Verify a signed access token

For a small first-party example, PyJWT can verify an HMAC-signed token. Keep the secret in environment configuration and use a fixed algorithm allowlist:

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

Create a local development key with Python using `python -c "import secrets; print(secrets.token_urlsafe(32))"`, then provide it through the `JWT_SECRET` environment variable. The key must never be committed to the repository. The example issuer and audience must match the trusted issuer configuration. Do not accept the token algorithm from an untrusted header as your verification policy.

An HMAC secret is shared by every service that verifies these tokens. For systems with several services, an identity provider commonly signs tokens with a private key and publishes public keys for verification. Follow the provider key rotation and issuer metadata.

## Turn a verified token into a current user

Only use identity claims after cryptographic and claim validation succeeds. Then load the current account and verify it is still active:

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

`find_active_user` represents a database lookup and should be implemented with the session dependency from the database chapter. Return only public account fields from the route.

## Check permissions for each operation

Authentication is not enough for access to a specific record. Check ownership or a permission rule every time a protected resource is read or changed:

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

Do not accept `owner_id` from the request body as proof of ownership. Read the authenticated identity from the verified user dependency, then check access in the database query or service operation.

For team and application roles, define the permissions the action needs. A user role sent by the client is just input; use the role loaded from trusted server-side data.

## Hash passwords only if the application owns them

When an application must store passwords, use `pwdlib` with Argon2 and store only the resulting hash:

~~~python
from pwdlib import PasswordHash

password_hasher = PasswordHash.recommended()


def hash_password(password: str) -> str:
    return password_hasher.hash(password)


def verify_password(password: str, stored_hash: str) -> bool:
    return password_hasher.verify(password, stored_hash)
~~~

Password hashes include the information needed for verification and should be stored in the user record. Verify a submitted password against the stored hash; do not decrypt it or compare plain text.

Sign-in endpoints also need rate limits, generic failure messages, secure recovery, and monitoring. Keep password and token values out of logs.

## Know what a JWT does not do

A signed JWT is usually readable by anyone who has the token. A signature lets a verifier detect changes and check who issued the token; it does not hide its contents.

Access tokens should have a limited lifetime. A signed token is not automatically revoked when a user is disabled, a role changes, or a key is compromised. Decide how the application handles revocation, short token lifetimes, refresh tokens, and key rotation.

## Practice

1. Read an `Authorization: Bearer` header and confirm that extraction alone does not authenticate it.
2. Verify a signed token with fixed algorithm, issuer, audience, and expiration checks.
3. Return `401` with a bearer challenge for an invalid token.
4. Load the account associated with the verified subject and reject disabled accounts.
5. Check resource ownership before returning a private record.
6. Hash and verify a password with Argon2 if the application stores passwords itself.

## Main references

- [FastAPI security reference](https://fastapi.tiangolo.com/reference/security/)
- [PyJWT documentation](https://pyjwt.readthedocs.io/en/stable/)
- [pwdlib repository](https://github.com/frankie567/pwdlib)
- [OWASP password storage guidance](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [OWASP OAuth 2.0 guidance](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html)
