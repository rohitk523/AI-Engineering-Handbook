# 07 — Authentication and Authorization in FastAPI ⭐

[← 06 FastAPI](06-fastapi-fundamentals.md) · [Index](00-INDEX.md) · Next: [08 Serving →](08-serving-llm-apps.md)

---

## 1. Vocabulary (get this exactly right in interviews)

| Term | Question it answers | HTTP failure |
|---|---|---|
| **Authentication (AuthN)** | *Who are you?* | **401** Unauthorized (really means "unauthenticated") |
| **Authorization (AuthZ)** | *What are you allowed to do?* | **403** Forbidden |
| **Session auth** | Server stores session, browser holds a cookie with the session ID | |
| **Token auth** | Client sends a token (often a JWT) in `Authorization: Bearer <token>` | |
| **OAuth 2.0 / 2.1** | Framework for *delegated* access (an app acts on a user's behalf) | |
| **OpenID Connect (OIDC)** | Identity layer on top of OAuth 2 ("Sign in with Google"): adds the ID token | |
| **JWT** | Signed JSON token: `header.payload.signature` (base64url). **Signed, not encrypted**, so anyone can read the payload | |

---

## 2. How FastAPI models security

FastAPI ships **security dependencies** in `fastapi.security` that read credentials from the request *and* document them in OpenAPI (the "Authorize" button in `/docs`):

- `OAuth2PasswordBearer(tokenUrl="token")`: reads `Authorization: Bearer ...`
- `APIKeyHeader(name="X-API-Key")`, `APIKeyCookie`, `APIKeyQuery`
- `HTTPBasic`, `HTTPBearer`
- `SecurityScopes` for OAuth2 scopes

They **only extract** the credential. Verifying it is your job, done in a dependency you write (`get_current_user`). Auth in FastAPI is just dependency injection ([06 §4](06-fastapi-fundamentals.md)).

---

## 3. Full JWT login flow (password grant, first-party app)

```
1. POST /token  (form: username, password)
2. Server: look up user → verify password hash → issue signed JWT (sub=user_id, exp=+15min)
3. Client stores token; sends  Authorization: Bearer <jwt>  on each request
4. Protected endpoint: dependency decodes + verifies JWT → loads user → endpoint runs
```

```python
# core/security.py
from datetime import datetime, timedelta, timezone
import jwt                                  # PyJWT
from pwdlib import PasswordHash             # current FastAPI docs recommend pwdlib (Argon2)

password_hash = PasswordHash.recommended()
SECRET_KEY = settings.jwt_secret            # from env/secret manager, NEVER hardcoded
ALGORITHM = "HS256"
ACCESS_TTL = timedelta(minutes=15)

def hash_password(plain: str) -> str:
    return password_hash.hash(plain)

def verify_password(plain: str, hashed: str) -> bool:
    return password_hash.verify(plain, hashed)

def create_access_token(user_id: str, scopes: list[str]) -> str:
    now = datetime.now(timezone.utc)
    payload = {"sub": user_id, "scopes": scopes, "iat": now, "exp": now + ACCESS_TTL}
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
```

```python
# auth/router.py
from typing import Annotated
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
import jwt

router = APIRouter()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

CREDENTIALS_ERROR = HTTPException(
    status_code=status.HTTP_401_UNAUTHORIZED,
    detail="Could not validate credentials",
    headers={"WWW-Authenticate": "Bearer"},
)

@router.post("/token")
async def login(form: Annotated[OAuth2PasswordRequestForm, Depends()], db: DB):
    user = await users_repo.get_by_username(db, form.username)
    # Same error for "no such user" and "wrong password", so attackers can't enumerate users.
    if user is None or not verify_password(form.password, user.password_hash):
        raise HTTPException(status_code=401, detail="Incorrect username or password")
    return {"access_token": create_access_token(str(user.id), user.scopes),
            "token_type": "bearer"}

async def get_current_user(token: Annotated[str, Depends(oauth2_scheme)], db: DB) -> User:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])  # verifies signature + exp
    except jwt.InvalidTokenError:
        raise CREDENTIALS_ERROR
    user = await users_repo.get(db, payload.get("sub"))
    if user is None or not user.is_active:
        raise CREDENTIALS_ERROR
    return user

CurrentUser = Annotated[User, Depends(get_current_user)]

@router.get("/me")
async def me(user: CurrentUser):
    return {"id": user.id, "username": user.username}
```

Notes:
- Always pass `algorithms=[...]` explicitly when decoding. Never accept `alg: none` or let the token choose.
- The tutorials used `python-jose` and `passlib` for years. Current docs moved to **PyJWT** and **pwdlib**, since those older libraries are poorly maintained.
- Hash with **Argon2id** (or bcrypt). Never SHA-256 or MD5 for passwords.

---

## 4. Authorization: roles, scopes and resource ownership

```python
def require_role(*roles: str):
    async def checker(user: CurrentUser) -> User:
        if user.role not in roles:
            raise HTTPException(status_code=403, detail="Forbidden")
        return user
    return checker

@router.delete("/orders/{order_id}", dependencies=[Depends(require_role("admin"))])
async def delete_order(order_id: int): ...
```

- **RBAC** (role-based): admin / staff / customer.
- **Scopes** (OAuth2): `orders:read`, `orders:write`, enforced with `Security(get_current_user, scopes=[...])` and `SecurityScopes`.
- **Resource-level checks (the most common real vulnerability, IDOR/BOLA):** a user is authenticated but requests *someone else's* order. Always filter by owner: `WHERE id = :id AND user_id = :current_user`.
- For complex policies: ABAC or a policy engine (OPA, Cerbos, Oso, OpenFGA/Zanzibar-style relationship checks).

---

## 5. Refresh tokens and logout

JWTs are **stateless**: you can't "delete" one before `exp`. Standard pattern:
- **Short-lived access token** (5–15 min) + **long-lived refresh token** (days), the refresh token stored server-side (DB/Redis) so it can be revoked.
- **Rotate** the refresh token on each use. If an old one is reused, assume theft and revoke the whole token family.
- Logout = revoke the refresh token (optionally deny-list the access token's `jti` in Redis until it expires).

### Where to store tokens in a browser
| Option | Risk |
|---|---|
| `localStorage` | Readable by any JS, so XSS steals it |
| **`HttpOnly; Secure; SameSite` cookie** | JS can't read it. Needs CSRF protection (SameSite + CSRF token for state-changing requests) |
| Memory only + refresh via cookie | Common SPA compromise |

---

## 6. API keys (machine-to-machine, and very common for AI APIs)

```python
import hashlib, hmac, secrets
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

def generate_api_key() -> tuple[str, str]:
    raw = "sk_live_" + secrets.token_urlsafe(32)        # show once to the user
    return raw, hashlib.sha256(raw.encode()).hexdigest() # store only the hash

async def get_api_client(key: Annotated[str | None, Depends(api_key_header)], db: DB):
    if not key:
        raise HTTPException(401, "Missing API key")
    digest = hashlib.sha256(key.encode()).hexdigest()
    client = await keys_repo.get_by_hash(db, digest)
    if client is None or client.revoked:
        raise HTTPException(401, "Invalid API key")
    return client
```

Store **hashes** (a high-entropy random key makes plain SHA-256 fine here, unlike passwords). Use a recognizable prefix so secret scanners can detect leaks. Support rotation, per-key scopes and per-key rate limits. Use `hmac.compare_digest` for any direct comparison.

---

## 7. Using an identity provider (what most companies actually do)

Rolling your own user database is often unnecessary. Common choices: **Auth0/Okta, AWS Cognito, Azure Entra ID, Google Identity, Keycloak (self-hosted), Clerk, Supabase Auth, WorkOS** (enterprise SSO).

Your FastAPI app then becomes an **OAuth resource server**:
1. Frontend logs in with the IdP (Authorization Code + **PKCE** flow).
2. Frontend sends the IdP's access token (JWT) to your API.
3. Your API verifies the signature with the IdP's public keys (**JWKS** endpoint, cached), and checks `iss`, `aud`, `exp`, and scopes.

```python
jwks_client = jwt.PyJWKClient(settings.jwks_url)

def verify_idp_token(token: str) -> dict:
    signing_key = jwks_client.get_signing_key_from_jwt(token)
    return jwt.decode(token, signing_key.key, algorithms=["RS256"],
                      audience=settings.api_audience, issuer=settings.issuer)
```

---

## 8. Auth for agents (the new part)

- **Agent acting for a user**: the agent should use *the user's* delegated, scoped token (OAuth on-behalf-of / token exchange), never a god-mode service account. This is how remote MCP servers handle auth (OAuth 2.1, [04](04-mcp-and-protocols.md)).
- **Least privilege per tool**: read-only scopes by default; write scopes need explicit consent.
- **Human approval** for irreversible actions (payments, sending email, deleting data).
- **Audit log** every tool call with user, agent run ID, args and result.
- Never put secrets in prompts. The model can leak anything in its context. Tools fetch secrets server-side.

---

## 9. Security checklist

- [ ] Secrets in env vars / a secret manager. Validated at startup
- [ ] Passwords: Argon2id/bcrypt. Login errors don't reveal whether the user exists
- [ ] JWT: explicit algorithm, `exp` checked, short TTL, `aud`/`iss` checked for IdP tokens
- [ ] HTTPS only. Cookies `HttpOnly; Secure; SameSite`
- [ ] CORS: explicit origin allow-list, never `*` with credentials
- [ ] Rate limit `/token` and login endpoints (brute force protection)
- [ ] Object-level authorization on every resource query
- [ ] Don't log tokens, passwords or API keys

---

## 10. Check yourself

- 401 vs 403: give an example of each.
- Why can't you log out a JWT, and what do you do about it?
- Walk through the Authorization Code + PKCE flow.
- How do you verify a JWT issued by Auth0 in FastAPI?
- How would you let an AI agent read a user's Google Calendar without giving it the user's password?
- What's IDOR and how does your code prevent it?
