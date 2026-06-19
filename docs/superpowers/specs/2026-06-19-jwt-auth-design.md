# JWT Auth with Refresh Tokens + Google & Microsoft OAuth — Design

**Date:** 2026-06-19
**Status:** Approved (design)
**Scope:** `hefest-api` authentication subsystem
**Covers:** [S1](../../criteria/grading-criteria.md#security-15-points) (password hashing) · [S2](../../criteria/grading-criteria.md#security-15-points) (input validation) · [S3](../../criteria/grading-criteria.md#security-15-points) (authorization) · [SC3](../../criteria/grading-criteria.md#scalability--design-15-points) (stateless horizontal scaling)

---

## 1. Summary

Authentication issues a short-lived **access token** (stateless JWT) plus a long-lived,
**server-side, rotating refresh token** (opaque, stored hashed in Postgres). Users authenticate
with email + password, **Sign in with Google**, or **Sign in with Microsoft** (Entra ID — for
Bulgarian `edu.mon.bg` government education accounts). In every case the backend mints **our own**
token pair — the provider's tokens are never used to call the Hefest API.

**Email verification gates local accounts:** a local registration is **inactive until the email is
confirmed** (verification link sent through the existing notification pipeline). An OAuth sign-in is
proof of email ownership, so SSO users are **active immediately**.

**Client:** a browser **SPA** today; a **mobile app is planned later**. Transport supports both — a
web client uses an httpOnly refresh cookie; a future mobile client uses a Bearer/body refresh token.
The refresh endpoint reads the token from **cookie or body**, so the mobile path needs no redesign.

OAuth providers live in a provider-agnostic `oauth_identities` table, so adding or re-prioritising a
provider later (e.g. making Microsoft primary) needs no schema change.

**Providers are individually toggleable.** A provider is *enabled* only when its config is present, so
running with **no OAuth apps set up at all** is the default — email/password auth always works, and
Google/Microsoft light up when (and only when) their env vars are filled in. A public
`GET /auth/providers` tells the frontend which login options to render.

Auth lives **inside the `api` process** (not a separate service). Token *validation* is stateless
in-process JWT decoding on every request; token *issuance* is low-frequency. The API therefore
scales horizontally by running more `api` replicas — this is the SC3 scalability story.

---

## 2. Token model

| Token | Type | Lifetime | Storage | Notes |
|---|---|---|---|---|
| Access | JWT, HS256 (`HEFEST_JWT_SECRET`) | **15 min** | none (stateless) | Claims: `sub` (user id, str), `role`, `iss="hefest"`, `aud="hefest-api"`, `iat`, `exp`, `type="access"` |
| Refresh | Opaque random (`secrets.token_urlsafe(32)`) | **14 days** | Postgres `refresh_tokens`, **SHA-256 hash only** | Single-use, rotated on every refresh |
| Email-verify | JWT, HS256 | **24 h** | none (stateless) | Claims: `sub`, `aud="hefest-verify"`, `exp`, `type="email_verify"` — embedded in the verification link |

**Why opaque + SHA-256 (not JWT, not bcrypt) for refresh:** refresh tokens are high-entropy random
strings, so a fast SHA-256 is safe at rest and allows an indexed lookup by hash. bcrypt would force a
table scan. The opaque token carries no claims and is meaningless if leaked from the DB (only the
hash is stored).

**`aud` claim:** access tokens are validated with `aud="hefest-api"`; the verify-email JWT uses a
distinct `aud="hefest-verify"` so a verify link can never be replayed as an API token, and a shared
secret across environments can't cross-validate.

**HS256 vs RS256:** HS256 is correct here because auth and the API are colocated in one process — any
verifier already holds the signing secret. If a separate service ever needs to *verify* tokens
without being able to *forge* them, migrate to RS256/ES256 with a published public key. (Algorithm is
read from config, so this is a config + key-distribution change, not a code rewrite.)

**Rotation:** every `POST /auth/refresh` invalidates the presented token (`revoked_at = now()`) and
issues a fresh pair, **atomically** (see §4).

**Reuse detection:** if a token that is already revoked is presented to `/auth/refresh`, the token
was stolen and replayed after the legitimate client already rotated it. The whole family is
compromised, so we revoke **all** of that user's refresh tokens and reject (`401
token_reuse_detected`), forcing a fresh login everywhere (aligns with RFC 6819).

**Config (`config.py`):**

```python
jwt_expire_minutes: int = 15              # was 60
jwt_audience: str = "hefest-api"          # new — validated on every access token
refresh_token_expire_days: int = 14       # new
email_verify_expire_hours: int = 24       # new

google_client_id: str = ""                # HEFEST_GOOGLE_CLIENT_ID
google_client_secret: str = ""            # HEFEST_GOOGLE_CLIENT_SECRET, .env only
google_redirect_uri: str = ""             # HEFEST_GOOGLE_REDIRECT_URI
microsoft_client_id: str = ""             # HEFEST_MICROSOFT_CLIENT_ID
microsoft_client_secret: str = ""         # HEFEST_MICROSOFT_CLIENT_SECRET, .env only
microsoft_tenant: str = ""                # HEFEST_MICROSOFT_TENANT (edu.mon.bg Entra tenant id)
microsoft_redirect_uri: str = ""          # HEFEST_MICROSOFT_REDIRECT_URI

frontend_oauth_success_url: str = ""      # 302 target after OAuth, e.g. https://app.hefest.bg/oauth
cors_origins: list[str] = []              # explicit allowed origins (credentials mode)
refresh_cookie_name: str = "hefest_refresh"
refresh_cookie_secure: bool = True        # False only in local dev over http
# (rate-limit settings already exist in config.py — reused here, see §7)
```

**Provider enablement (no toggle table needed):** a provider is *enabled* iff **all** its required
settings are non-empty — Google: `client_id` + `client_secret` + `redirect_uri`; Microsoft: those
plus `tenant`. The full set of providers the code knows how to speak to is a constant; the *enabled*
subset is derived from config — both feed `GET /auth/providers` (§4):

```python
SUPPORTED_OAUTH_PROVIDERS: Final = ("google", "microsoft")   # everything the code can do

@property
def enabled_oauth_providers(self) -> list[str]:              # the configured subset
    out: list[str] = []
    if self.google_client_id and self.google_client_secret and self.google_redirect_uri:
        out.append("google")
    if all((self.microsoft_client_id, self.microsoft_client_secret,
            self.microsoft_redirect_uri, self.microsoft_tenant)):
        out.append("microsoft")
    return out
```

Leaving the env vars blank is the intended way to run without setting up the OAuth apps. Local
password auth is always on. (An explicit `HEFEST_<PROVIDER>_ENABLED=false` hard-off, even when
configured, is optional — §8.)

### Configuration reference

All variables are read with the `HEFEST_` prefix (e.g. `jwt_secret` → `HEFEST_JWT_SECRET`). Secrets
live only in `.env` (gitignored) and are never logged.

| Env var | Required? | Default | Purpose |
|---|---|---|---|
| `HEFEST_JWT_SECRET` | **yes (prod)** | `change-me-in-production` | HS256 signing key for access & verify JWTs. |
| `HEFEST_JWT_ALGORITHM` | no | `HS256` | Signing algorithm; switch point for the RS256 path (§2). |
| `HEFEST_JWT_AUDIENCE` | no | `hefest-api` | Validated `aud` on every access token. |
| `HEFEST_JWT_EXPIRE_MINUTES` | no | `15` | Access-token lifetime. |
| `HEFEST_REFRESH_TOKEN_EXPIRE_DAYS` | no | `14` | Refresh-token lifetime. |
| `HEFEST_EMAIL_VERIFY_EXPIRE_HOURS` | no | `24` | Verification-link lifetime. |
| `HEFEST_REFRESH_COOKIE_NAME` | no | `hefest_refresh` | Name of the httpOnly refresh cookie. |
| `HEFEST_REFRESH_COOKIE_SECURE` | no | `true` | Set `false` only for local http dev. |
| `HEFEST_FRONTEND_OAUTH_SUCCESS_URL` | OAuth only | `""` | 302 target after a successful OAuth login. |
| `HEFEST_CORS_ORIGINS` | web SPA | `[]` | Allowed browser origins (credentials mode). |
| `HEFEST_GOOGLE_CLIENT_ID` | Google | `""` | **Enables Google** when all three Google vars are set. |
| `HEFEST_GOOGLE_CLIENT_SECRET` | Google | `""` | Google OAuth client secret. |
| `HEFEST_GOOGLE_REDIRECT_URI` | Google | `""` | Must exactly match the Google console (trailing slash matters). |
| `HEFEST_MICROSOFT_CLIENT_ID` | Microsoft | `""` | **Enables Microsoft** when all four MS vars are set. |
| `HEFEST_MICROSOFT_CLIENT_SECRET` | Microsoft | `""` | Entra app client secret. |
| `HEFEST_MICROSOFT_TENANT` | Microsoft | `""` | `edu.mon.bg` Entra tenant id — restricts sign-in to that org. |
| `HEFEST_MICROSOFT_REDIRECT_URI` | Microsoft | `""` | Must exactly match the Azure console. |

A provider's whole group is **all-or-nothing**: set every var in the group to enable it, or leave the
group blank to run without it. Partial config (e.g. a Google client id but no secret) leaves the
provider **disabled** and reported as `available: false`.

---

## 3. Data model changes

### 3.1 `users` table (one migration)

- `password_hash` → **nullable** (an OAuth-only user has no password).
- Add `email_verified_at` → `DatetimeField(null=True)`. `NULL` = unverified/inactive. Set on
  successful email verification, or immediately on first OAuth sign-in.
- No per-provider columns — provider identities live in `oauth_identities` (§3.2).

### 3.2 New `oauth_identities` table

Provider-agnostic link table so any number of OAuth providers map to one Hefest user, and one user
can link several providers (local + Google + Microsoft).

```
oauth_identities
  id          uuid PK
  user_id     uuid FK -> users (indexed, on_delete=CASCADE)
  provider    text                 # 'google' | 'microsoft'
  subject     text                 # provider's stable subject id (OpenID.id)
  email       text                 # email at the provider; refreshed on each login (audit)
  created_at  timestamptz
  UNIQUE (provider, subject)
```

We key on `(provider, subject)`, **not email**: emails change or get reassigned; the provider subject
is stable. `email` is updated on each login to track the current provider contact. Adding a provider
is **data-model-free**.

### 3.3 New `refresh_tokens` table

```
refresh_tokens
  id           uuid PK
  user_id      uuid FK -> users (indexed, on_delete=CASCADE)
  token_hash   text UNIQUE          # sha256 hex of the opaque token (UNIQUE -> B-tree index)
  expires_at   timestamptz
  revoked_at   timestamptz NULL     # set on rotation, logout, or family revocation
  created_at   timestamptz
```

Valid iff `revoked_at IS NULL AND expires_at > now()`. Tortoise model → tracked by `makemigrations`.
Expired rows are harmless (rejected on read); pruning is optional (§8).

---

## 4. Endpoints

All under the `auth` router. Errors use the project envelope `{ "detail", "code" }`.

| Method | Path | Auth | Behavior |
|---|---|---|---|
| `GET`  | `/auth/providers` | public | Frontend capability discovery — which login methods are available (see below). |
| `POST` | `/register` | public | Create **unverified** student (hash password, `email_verified_at=NULL`) + enqueue verification email (outbox, same transaction). Returns **201, no tokens**. In `HEFEST_ENV=dev` the response also includes a `verify_token` field for local testing — **never present in any other environment** (see implementation note below). |
| `POST` | `/auth/verify-email` | verify JWT (link) | Decode the `email_verify` JWT; set `email_verified_at=now()` (idempotent). Activates the account. |
| `POST` | `/login` | public | Verify bcrypt password. If `email_verified_at IS NULL` → `403 email_not_verified`. Else issue access (JSON) + refresh (cookie **or** body per transport, §5). |
| `POST` | `/auth/refresh` | refresh token (cookie or body) | Look up by SHA-256 hash. **Atomic** guarded update `… SET revoked_at=now() WHERE id=? AND revoked_at IS NULL`; if it updated a row → issue new pair. If the row exists but was **already revoked** → reuse detected: revoke the whole family, `401 token_reuse_detected`. |
| `POST` | `/auth/logout` | refresh token (cookie or body) | Revoke that row; clear the refresh cookie. |
| `POST` | `/auth/logout-all` | **access Bearer _or_ refresh token** | Revoke all refresh rows for the user. Accepts a refresh token in the body too, so a user with an expired access token can still log out everywhere. |
| `GET`  | `/auth/google/login` | public | `fastapi-sso` → 302 to Google. |
| `GET`  | `/auth/google/callback` | public | Verify → find-or-create user → **302 redirect to frontend** with access token + refresh cookie. |
| `GET`  | `/auth/microsoft/login` | public | `fastapi-sso` → 302 to Microsoft (single Entra tenant). |
| `GET`  | `/auth/microsoft/callback` | public | Same as Google callback, `provider="microsoft"`. |

The `*/login` and `*/callback` pairs share one generic implementation (§5) — only the SSO client and
`provider` literal differ.

**Implementation note — `verify_token` in `/register` response (dev only):**

Until the notification pipeline delivers verification emails, `POST /register` exposes the
`email_verify` JWT directly in the response body — **exclusively when `HEFEST_ENV=dev`**:

```json
{
  "message": "registered; check your email to verify your account",
  "verify_token": "eyJ..."   ← only present when HEFEST_ENV=dev
}
```

The gate is `if settings.env == "dev"` in `routers/auth.py`. In all other environments (`staging`,
`production`, …) only `{"message": "..."}` is returned. This prevents the verification bypass from
leaking to non-dev deployments: a caller cannot self-verify without actual email access unless the
server is explicitly running in dev mode. When the notification pipeline is wired (§9 follow-up), the
`verify_token` dev shortcut should be removed entirely.

**Provider discovery (`GET /auth/providers`)** — lists **every theoretically-supported provider**
(`SUPPORTED_OAUTH_PROVIDERS`, §2), each marked `available` according to
`settings.enabled_oauth_providers` — the single source of truth, so it can never disagree with which
routes actually work:

```json
{
  "password": { "available": true },
  "providers": [
    { "name": "google",    "available": true,  "login_url": "/auth/google/login" },
    { "name": "microsoft", "available": false, "login_url": null }
  ]
}
```

Unconfigured providers stay in the list with `available: false` and `login_url: null`, so the
frontend can render them as disabled/"unavailable" buttons (e.g. greyed out) rather than hiding them.
The frontend renders all login options from this response instead of hard-coding them.

**Disabled-provider routes:** the `*/login` and `*/callback` routes for a provider that isn't enabled
respond `404 { "detail": "...", "code": "sso_provider_disabled" }` (a single guard derived from the
same enabled set) — so a stale bookmark or hand-crafted request fails clearly rather than 500-ing on a
missing client.

**Concurrency:** `/auth/refresh` rotation and family-revocation run inside a DB transaction; rotation
uses the atomic guarded `UPDATE … WHERE revoked_at IS NULL` (check affected rowcount) so 10 concurrent
replays of one token cannot each mint a new pair — only the first update wins, the rest see an
already-revoked row and trigger reuse detection. (Same `FOR UPDATE`/atomic discipline as the
registration transaction.)

### Transport

- **Web (cookie mode, default):** access token in the JSON body (SPA holds it in memory); refresh
  token in a `Secure; HttpOnly; SameSite=Strict; Path=/auth` cookie. OAuth callbacks 302-redirect to
  `frontend_oauth_success_url` with the access token in the URL **fragment** (`#access_token=…`,
  invisible to server logs) and the refresh cookie set.
  - **Frontend requirement (review #2):** on the OAuth-success route the SPA must read the token into
    memory and **immediately** call `window.history.replaceState()` to scrub the `#access_token=…`
    fragment from the address bar, so it can't leak via history, shoulder-surfing, screen-share, or
    URL-reading extensions. (A future hardening that removes the token from the URL entirely — a
    one-time code exchanged at a `POST /auth/oauth/exchange` endpoint — is noted in §8.)
- **Mobile (Bearer mode, future):** client opts in (e.g. `X-Auth-Transport: bearer` on login); the
  refresh token is returned in the JSON body instead of a cookie and sent back in the body on
  `/auth/refresh`. The endpoint already reads refresh from **cookie or body**, so this is additive.

JSON body shape (access token always; refresh only in Bearer mode):

```json
{ "access_token": "eyJ...", "token_type": "bearer", "expires_in": 900 }
```

---

## 5. OAuth providers (Google & Microsoft, via `fastapi-sso`)

Redirect + callback, with `fastapi-sso` owning each handshake (state/PKCE/code-exchange). We ignore
the provider's tokens and mint our own pair. Both providers share one generic callback.

SSO clients are constructed **only for enabled providers** (`settings.enabled_oauth_providers`, §2),
so an unconfigured provider instantiates nothing and a disabled `*/login`/`*/callback` is guarded to
`404 sso_provider_disabled` (§4).

```python
# built lazily per enabled provider; absent providers are never instantiated
google_sso = GoogleSSO(google_client_id, google_client_secret, google_redirect_uri, use_state=True)
microsoft_sso = MicrosoftSSO(
    microsoft_client_id, microsoft_client_secret,
    tenant=microsoft_tenant,                       # single-tenant: only edu.mon.bg accounts
    redirect_uri=microsoft_redirect_uri, use_state=True,
    scope=["openid", "email", "profile"],          # explicit for Entra
)

async def _oauth_callback(request: Request, response: Response, sso, provider: str) -> RedirectResponse:
    async with sso:
        openid = await sso.verify_and_process(request)          # -> OpenID
    user = await find_or_create_oauth_user(provider, openid)    # services/auth.py
    access = create_access_token(user)
    set_refresh_cookie(response, issue_refresh_token(user))     # web; Bearer mode returns body
    return RedirectResponse(f"{frontend_oauth_success_url}#access_token={access}")

@router.get("/auth/google/callback")
async def google_callback(request: Request, response: Response):
    return await _oauth_callback(request, response, google_sso, "google")
# /auth/microsoft/callback is the same body with microsoft_sso and "microsoft".
```

**OpenID mapping:** an `oauth_identities` row is `provider`, `subject = openid.id`,
`email = openid.email`; the linked `User` gets `full_name = openid.display_name`, role = `student`
(organizers stay seed-only). Use `openid.model_dump()` (Pydantic v2), never `.dict()`.

**`find_or_create_oauth_user(provider, openid)` logic:**

Run inside a transaction (the `users.email` UNIQUE constraint is the backstop — on a concurrent
`IntegrityError`, re-fetch and re-evaluate from step 1):

1. Identity exists for `(provider, openid.id)` → return its user; refresh stored `email` if changed.
2. Else look up a user by `openid.email`:
   - **Verified** local user (`email_verified_at IS NOT NULL`) **and** provider asserts the email is
     verified → **auto-link**: insert an identity row, return that user.
   - **Unverified** local user (`email_verified_at IS NULL`) → **take over the dormant row**, because
     the original registrant never proved ownership and the provider just did: set
     `email_verified_at = now()`, **`password_hash = NULL`** (kills the squatter's password),
     `full_name = openid.display_name`; insert the identity row; return that user.
   - No user with that email → create a new `student` user (`email_verified_at = now()`) + identity row.

> **Account-takeover guard (review #1):** auto-link only ever attaches to a *verified* local account.
> The dangerous case — a squatter who registered someone else's email locally — leaves an
> **unverified** row; on the real owner's OAuth sign-in we **take that row over and null its
> password**, so the squatter loses all access. This is safe because an unverified account can never
> have authenticated (register issues no tokens, login is blocked until verified), so the row owns no
> events or registrations — there is nothing to steal, only a name to claim. Creating a *new* user on
> an email collision is **avoided** precisely because `users.email` is `UNIQUE` and would otherwise
> raise an `IntegrityError`.

**Provider verification signal:** Google → `email_verified == true`. Microsoft Entra single-tenant →
a successful tenant login is authoritative for `@edu.mon.bg` (the org owns the domain); optionally
also allowlist the email domain as defence in depth.

**CSRF / state:** `use_state=True` — `fastapi-sso` generates and verifies the `state`; the frontend
must **not** supply it. Redirect URIs must be **exact** matches in the Google/Azure consoles
(trailing slashes matter).

**Library risk** (`fastapi-sso` churns across minors; Entra less battle-tested than Google):

- **Pin exactly** in `pyproject.toml` (`fastapi-sso==<latest>`), not a range.
- **Spike the Entra flow first** (real tenant, real callback) before committing. The library is used
  **only** inside `_oauth_callback`/the OAuth routes; if Entra misbehaves, swap the Microsoft branch
  to `authlib` + `httpx` behind the same helper — single-helper blast radius.

---

## 6. Module layout (fills the empty `hefest/` stubs)

| File | Responsibility |
|---|---|
| `models/refresh_token.py` | `RefreshToken` Tortoise model (§3.3) |
| `models/oauth_identity.py` | `OAuthIdentity` Tortoise model (§3.2) |
| `schemas/auth.py` | `RegisterRequest` (with `PasswordPolicy` field validator), `LoginRequest`, `VerifyEmailRequest`, `TokenResponse` — Pydantic v2 (**S2**) |
| `services/auth.py` | `hash_password`/`verify_password` (passlib bcrypt, **S1**); `create_access_token`; `issue_refresh_token`; `rotate_refresh_token` (atomic + reuse detection); `revoke`; `revoke_all_for_user`; `create_email_verify_token`/`consume_email_verify_token`; `find_or_create_oauth_user`; `set_refresh_cookie` |
| `services/notifications.py` (existing pipeline) | register enqueues the verification email as an outbox row |
| `routers/auth.py` | All endpoints in §4 + the generic `_oauth_callback` helper (§5) + `GET /auth/providers` + the `sso_provider_disabled` guard, all reading `settings.enabled_oauth_providers` |
| `routers/deps.py` | `get_current_user` (decode + validate `aud`, load `User`) + `require_role(*roles)` — injected by every protected route (**S3**, **CQ4**); plus a `rate_limit(...)` dependency wrapping the existing config |

```python
# routers/deps.py — reuse that satisfies S3 across every protected route
async def get_current_user(token: str = Depends(oauth2_scheme)) -> User: ...   # validates aud="hefest-api"
def require_role(*roles: UserRole) -> Callable: ...   # UserRole is already a StrEnum
```

`CORSMiddleware` is registered in `main.py` with `allow_origins=cors_origins`,
`allow_credentials=True` (required for the refresh cookie).

---

## 7. Security notes (grading map)

- **S1** — passwords hashed with passlib **bcrypt, cost factor 12**; `password_hash` nullable for
  OAuth-only users. `PasswordPolicy` validator enforces a **minimum length (≥ 12)**; a
  common-password blacklist (`zxcvbn` / top-1000) is optional polish (§8).
- **S2** — all auth inputs validated via Pydantic v2; tokens and identities looked up by
  parameterised values, never concatenated. **Rate limiting** reuses the existing `config.py`
  settings (`rate_limit_login_*`, `rate_limit_register_*`, plus the global limiter) via a
  `rate_limit` dependency on `/login`, `/register`, and `/auth/refresh` (per-IP, and per-account on
  login).
- **S3** — `get_current_user` + `require_role` enforce role/ownership on every protected route;
  access tokens are short-lived (15 min) and carry a validated `aud`; refresh tokens are revocable
  (logout / logout-all-via-refresh / rotation) with reuse detection that revokes the whole family;
  unverified local accounts cannot authenticate; Microsoft sign-in is restricted to the `edu.mon.bg`
  Entra tenant.
- **SC3** — access-token validation is stateless in-process JWT decoding → `api` scales
  horizontally; refresh state is shared Postgres reachable by all replicas. HS256→RS256 migration
  path documented (§2) for any future external verifier.
- **Browser security** — refresh token lives in a `HttpOnly; Secure; SameSite=Strict` cookie (not
  JS-readable → XSS can't exfiltrate it); access token is short-lived and held in memory. `SameSite=Strict`
  is the CSRF defence for the cookie; CORS is locked to explicit `cors_origins` with credentials.
- Google and Microsoft `client_secret`s and the JWT secret live only in `.env` (gitignored); never
  logged.

---

## 8. Optional polish (implement only if time permits)

1. **Common-password blacklist** — `zxcvbn` or a top-1000 list in the `PasswordPolicy` validator.
2. **Expired/unverified row pruning** — periodic delete of `refresh_tokens` past `expires_at` and of
   stale unverified `users` (`email_verified_at IS NULL AND created_at < now() - interval`).
3. **Resend verification email** — `POST /auth/resend-verification`.
4. **Full mobile Bearer mode** — the `X-Auth-Transport` selector and body-refresh path are seamed in
   §4/§5; finishing and testing the mobile flow is deferred until the app exists.
5. **Microsoft as primary** — purely a UX-ordering change; `oauth_identities` already supports it.
6. **Explicit provider hard-off** — a `HEFEST_<PROVIDER>_ENABLED=false` flag to disable a provider
   even when its config is present (e.g. temporarily). Default enablement is config-presence (§2).
7. **One-time-code OAuth exchange** — instead of the access token in the URL fragment, the callback
   redirects with a short-lived single-use code the SPA exchanges via `POST /auth/oauth/exchange`.
   Removes the token from the URL entirely; supersedes the §4 fragment-scrub requirement when built.

---

## 9. Docs to update after implementation

- `architecture/index.md` — stack table says Auth is "**Stateless**"; update to *access-stateless,
  refresh-stateful, email-verified*. Add `fastapi-sso`, Google + Microsoft (Entra) OAuth, and the
  email-verification step to the stack.
- `architecture/api.md` — add `/auth/providers`, `/auth/verify-email`, `/auth/refresh`,
  `/auth/logout`, `/auth/logout-all`, `/auth/google/login`, `/auth/google/callback`,
  `/auth/microsoft/login`, `/auth/microsoft/callback`; note the 201-no-tokens register, the
  `403 email_not_verified` login case, `sso_provider_disabled` on disabled providers, access-token
  TTL, and cookie transport. SSO availability is exposed via `/auth/providers`, **not** `/health` or
  `/ready` (those stay ops-only liveness/readiness).
- `architecture/data-model.md` — add `oauth_identities` and `refresh_tokens`; mark
  `users.password_hash` nullable and add `users.email_verified_at`.
- **`hefest-api/.env.example`** — add every variable from the §2 configuration reference, grouped
  (core JWT / cookies / CORS / Google / Microsoft) with the same short comments, and the OAuth groups
  left **blank** so a fresh checkout runs with local-password auth only and both SSOs reported
  `available: false`.
- **`architecture/` config page** — surface the §2 configuration-reference table in the docs site
  (which vars enable which provider, all-or-nothing groups, secrets-in-`.env`-only) so operators can
  configure auth without reading the spec.

---

## 10. Review resolutions

| # | Item | Outcome |
|---|---|---|
| 1 | OAuth email auto-link takeover | **Fixed** — auto-link only to a *verified* local account; email verification added (§3.1, §5). |
| 1b | Unverified-email collision crashes on `users.email` UNIQUE | **Fixed** — take over the dormant unverified row (null its password) instead of creating a new user; transaction + IntegrityError backstop (§5). |
| 2b | Access token leaks via URL fragment | **Fixed** — frontend `replaceState` scrub requirement; optional one-time-code exchange (§4, §8). |
| — | Concurrency in reuse detection | **Fixed** — atomic guarded update in a transaction (§4). |
| — | XSS vs CSRF (SPA) | **Addressed** — refresh in httpOnly/SameSite cookie, access in memory (§4, §7). |
| 2 | Rate limiting "missing" | **Corrected** — already in `config.py`; now referenced and applied (§7). |
| 3 | HS256 limitation | **Documented** — RS256 migration note (§2). |
| 4 | logout-all needs valid access token | **Fixed** — also accepts a refresh token (§4). |
| 5 | Verify Entra support | **Verified** in lib docs; explicit scopes + spike + authlib fallback (§5). |
| 6 | `aud` claim | **Added** — `aud="hefest-api"` (§2). |
| 7 | Password rules | **Added** — min length + bcrypt cost 12; blacklist optional (§7, §8). |
| 8 | "Microslop" in URL path | **Rejected** — fabricated; spec uses `/auth/microsoft/...` throughout. |
| 9 | OAuth callback transport | **Fixed** — 302 redirect to frontend (§4, §5). |
| 10 | CORS + state CSRF | **Documented** — CORS origins, `use_state`, exact redirect URIs (§5, §6). |
| 11 | `token_hash` index | **Non-issue** — `UNIQUE` emits a B-tree index (§3.3). |
| 12 | Prune expired tokens | **Optional** — §8. |
| 13 | `require_role`/`UserRole` enum | **Non-issue** — already a `StrEnum` (§6). |
| 14 | `oauth_identities.email` updates | **Decided** — refreshed on each login (§3.2, §5). |
