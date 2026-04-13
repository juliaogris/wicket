# Design: wicket v0

Domain-gated static file server. Serves static files behind Google
OAuth, restricted to a configured set of Google Workspace domains and
individual email addresses. Designed for deployment on Google Cloud
Run.

## Architecture

Wicket's core is an Axum auth middleware. Everything else is pluggable
around it.

**Auth middleware (the library):** an async middleware function
(`axum::middleware::from_fn`) that handles Google OAuth, session
cookies, and access gating. It can wrap any Axum router. This is the
thing other projects can depend on.

**Static file backend (one possible inner service):** serves files
from a local directory. This sits behind the auth middleware. It could
be swapped for a reverse proxy, an API, or anything else that speaks
HTTP.

**The binary (thin wiring):** reads config from environment variables
(v0) or a YAML config file (v1), picks a backend, wraps it with the
auth middleware, and starts the server.

This means wicket is a single crate that exposes both a library and a
binary. The library owns auth. The binary owns the specific use case
of serving static files behind that auth.

The same structure supports the future path and method access rules -
the rule engine replaces the simple domain/email check inside the
auth middleware, and nothing downstream changes.

When hearth (or another service) needs wicket as a library, the plan
is in-process composition: hearth exposes its `Router`, wicket wraps
it. No reverse proxy, no sidecar. Before that happens, split the
crate into a workspace: `wicket-auth` (the middleware, sessions,
OAuth) and `wicket` (the static file server binary, depends on
`wicket-auth`).

## Request flow

1. Client requests any path.
2. Auth middleware checks for a valid session cookie.
3. No valid session: redirect to `/auth/login`, which starts the
   Google OAuth flow.
4. User authenticates with Google, Google redirects back to
   `/auth/callback` with an authorization code.
5. Wicket exchanges the code for tokens, verifies the ID token JWT
   (signature via Google's JWKS, issuer, audience, expiry), and checks
   the `hd` claim and `email` claim against the allow-list.
6. On success, wicket issues a fresh signed session cookie (never
   reuses a pre-auth cookie) and redirects the user to their original
   destination.
7. Subsequent requests: middleware verifies the cookie signature and
   expiry, then passes the request through to the inner service with
   an `AuthUser` extension (email, domain) inserted into the request.

The inner service reads the authenticated user via
`Extension(user): Extension<AuthUser>`. This type is the library's
public API.

## hd claim and access gating

The `hd` (hosted domain) claim in Google's ID token identifies the
user's Google Workspace domain. Personal Gmail accounts have no `hd`
claim.

Access gating rules:

- If the token has an `hd` claim and it matches a configured domain,
  allow.
- If the token's `email` claim matches a configured email address,
  allow.
- Otherwise, reject. Show a minimal "access denied" page with a link
  to retry with a different account.
- A missing `hd` claim is never treated as a pass-through. It means
  the user is on a personal Google account and must match by email.

Additional layer: if the GCP project belongs to the same Workspace
org, setting the OAuth consent screen to "Internal" blocks non-domain
users at Google's end before they even get a token. This is a
belt-and-suspenders measure, not a substitute for server-side
verification.

## Reserved routes

- `GET /auth/login` - start OAuth flow
- `GET /auth/callback` - OAuth callback
- `GET /auth/logout` - clear session, redirect to `/`

The auth middleware intercepts these paths before dispatch to the
inner service. This is the only model that works when the inner
service is opaque (like hearth's API).

All other routes are forwarded to the inner service (static files in
the default binary).

## OAuth

Standard Authorization Code flow with Google's OAuth 2.0 endpoints.
The `hd` parameter in the authorization URL hints Google to show only
accounts from configured domains. The `hd` claim in the returned
ID token is verified server-side - this is the actual enforcement.

PKCE is not needed. Wicket is a confidential server-side client with
a client secret. The code exchange happens server-to-server.

A `nonce` is included in the auth request, stored alongside the state
in a short-lived cookie, and verified in the ID token. This prevents
replay attacks where an attacker captures a valid ID token and reuses
it within its validity window.

CSRF protection: a random `state` parameter (cryptographically random,
not a timestamp or UUID) is stored in a short-lived cookie before the
redirect to Google, and verified on callback. The state cookie is
`SameSite=Lax`, `HttpOnly`, with a 10-minute TTL. It is cleared after
the callback regardless of outcome (including error paths).

## Sessions

Stateless signed cookies. No server-side session store.

The cookie contains the user's email, domain, and expiry. It is signed
with HMAC-SHA256 using a secret from config. It is `HttpOnly`,
`Secure` (omitted when bound to localhost in dev mode), `SameSite=Lax`,
`Path=/`. The `Domain` attribute is not set (omitting it scopes the
cookie to the exact host, which is tighter than setting it).

Session duration is configurable (default 8 hours). No refresh token
flow for v0 - expired sessions go through the OAuth flow again, which
is typically skip-through for returning Google users.

**Limitation:** no server-side session store means no way to revoke a
session short of rotating the signing secret (which invalidates all
sessions). The blast radius of a compromised session is the full TTL.
This is acceptable for v0 because the threat model is limited: users
are already members of a configured domain or explicit allow-list.
If revocation becomes necessary, the lightest option is a short TTL
(1-2h) plus re-auth, or a small revocation list in Cloud Firestore.

**Dev mode:** when `WICKET_LISTEN_ADDR` binds to `127.0.0.1` or
`::1`, the `Secure` flag is omitted so cookies work over plain HTTP.
A warning is logged at startup. If `WICKET_LISTEN_ADDR` is not
loopback but `Secure` would be omitted, refuse to start.

## JWKS caching

Google's public signing keys are fetched from their JWKS endpoint and
cached in memory with a fixed 1-hour TTL. If token verification
encounters an unknown key ID (key rotation), the cache is evicted and
keys are re-fetched once. This handles rotation without parsing
`Cache-Control` headers.

The cache is an `Arc<RwLock<(Vec<Jwk>, Instant)>>` shared across
handlers via Axum state.

## Static file serving

Files are served from a local directory pointed to by an environment
variable. In the Docker image, the directory is copied alongside the
binary.

SPA fallback (serve `index.html` for unmatched paths) is configurable
and off by default. It belongs to the file server config, not the auth
config - when wicket wraps a non-SPA service (like hearth's API), SPA
fallback would be wrong.

Embedded static files via `rust-embed` (single-binary deployment) are
a future addition.

## Configuration

v0 uses environment variables:

- `WICKET_CLIENT_ID` - Google OAuth client ID (required)
- `WICKET_CLIENT_SECRET` - Google OAuth client secret (required)
- `WICKET_DOMAIN` - allowed Workspace domain (required)
- `WICKET_EMAILS` - comma-separated allow-listed email addresses
  (optional, for personal Gmail accounts)
- `WICKET_SESSION_SECRET` - HMAC signing key, base64-encoded, at
  least 32 bytes of raw entropy (required). On Cloud Run, use
  `--set-secrets` to source this from Secret Manager rather than
  storing it in the service configuration.
- `WICKET_SESSION_TTL` - session duration, e.g. `8h`, `30m`
  (default `8h`)
- `WICKET_LISTEN_ADDR` - bind address (default `0.0.0.0:8080`)
- `WICKET_STATIC_DIR` - directory to serve (required)
- `WICKET_SPA_FALLBACK` - serve `index.html` for unmatched paths
  (default `false`)

Config is split into two structs: auth config (client ID/secret,
domain, emails, session secret, session TTL) and file server config
(static dir, SPA fallback). When hearth uses the auth layer as a
library, it only sees the auth config.

## v1 config file

Replace the env vars with a YAML config file (`WICKET_CONFIG` env var
points to the file path). The config file is the foundation for
multi-provider auth and the path-based rules engine.

```yaml
listen: 0.0.0.0:8080
session:
  secret: base64-encoded-secret
  ttl: 8h

allow:
  google:
    domains:
      - goteleport.com
    emails:
      - julia.ogris@gmail.com
  github:
    orgs:
      - gravitational
    users:
      - juliaogris

static:
  dir: ./public
  spa_fallback: false
```

Each identity provider gets its own namespace under `allow:` with
provider-specific concepts (domains/emails for Google, orgs/users for
GitHub). Adding a provider is a new key under `allow:`, never a
schema change. Adding a mechanism within a provider (e.g. Google
groups) is a new key under that provider.

The path-based rules engine (future) builds on this same file:

```yaml
rules:
  - path: /admin/**
    if: 'contains(user.roles, "admin")'
  - path: /data/user/{USER_ID}/**
    methods: [GET, PUT]
    if: 'user.name == USER_ID'
  - path: /public/**
    methods: [GET]
```

## Deployment

Multi-stage Docker build: compile with the official Rust image, copy
the binary and static directory into a minimal distroless runtime
image.

Cloud Run provides custom domain support with managed TLS, horizontal
scaling, and scale-to-zero.

## Build tooling

Uses [just](https://github.com/casey/just) for build recipes. The
justfile will be introduced when there is something to build, not
before.

## Build order

Each step is independently testable and committable.

1. **Skeleton** - `cargo init`, dependencies, Axum hello-world with
   `/health` endpoint. Add `cargo clippy -- -D warnings` to the build
   from the start.
2. **Config** - environment variable parsing into auth config and file
   server config structs, passed as Axum state. Do this early so
   nothing is hardcoded in later steps.
3. **Session** - cookie signing and verification (HMAC-SHA256). This
   step introduces byte slices (`&[u8]`, `Vec<u8>`, base64 encoding)
   which will feel unfamiliar coming from Go. Write `#[test]`s for
   the sign/verify round-trip.
4. **Auth middleware** - an `axum::middleware::from_fn` function that
   checks session cookies and redirects unauthenticated requests.
   At this point the site is locked - every request redirects.
   Hard-code a fake session cookie in dev mode so you can verify
   the middleware logic without a real OAuth flow.
5. **OAuth flow** - login, callback, logout routes. JWKS fetching
   and caching. Nonce and state verification. This unlocks the site
   for allowed users.
6. **Static file serving** - serve files from a directory. Plug it
   in as the inner service behind the auth middleware.
7. **Dockerfile and Cloud Run** - containerise, deploy, verify
   end-to-end.

## Dependencies

Core: `axum`, `tokio`, `tower-http`, `reqwest`, `jsonwebtoken`,
`serde`, `thiserror`, `cookie`, `ring`, `base64`, `tracing`,
`tracing-subscriber`.

Using `ring` for HMAC-SHA256 instead of separate `hmac` + `sha2`
crates. `ring` is what production Rust code uses and provides a
single API for HMAC, random bytes, and key derivation.

Dropped from v0: `rust-embed` (directory serving is enough),
`rand` (ring handles random bytes).

## Open questions

- **SPA fallback**: should it be opt-in via env var (current design)
  or auto-detected from the presence of `index.html`?
- **Multiple domains in v0**: `WICKET_DOMAIN` is a single value. If
  multiple domains are needed before the v1 config file, use a
  comma-separated list.

## Future extensions

Features deferred from v0.

**YAML config file.** Replace env vars with a config file that
supports multi-provider auth (Google, GitHub) and per-provider
allow-lists. See "v1 config file" above.

**GitHub OAuth.** A second identity provider. Each provider
implements a common trait for token verification and claim
extraction. The `allow.github` config namespace supports orgs and
individual usernames.

**Embedded static files.** Bake files into the binary at compile
time with `rust-embed` for single-binary deployment.

**Cookie encryption.** Upgrade from HMAC signing to AES-256-GCM
encryption if the session payload grows beyond email and domain
(e.g. roles, group membership). Signing is sufficient when the
payload contains only non-sensitive data the user already knows.

**Session revocation.** A server-side revocation list (Cloud
Firestore or Redis) for revoking individual sessions without
rotating the signing secret.

**Path and method access rules.** Identity-aware HTTP gateway with
per-path rules, inspired by Firebase security rules. First match
wins, unmatched requests are denied. Path variables like `{USER_ID}`
are captured and available in `if:` expressions.

**Crate split.** Before hearth takes a library dependency on wicket,
split into a workspace: `wicket-auth` (middleware, sessions, OAuth)
and `wicket` (static file server binary).

**tower::Layer extraction.** Replace `middleware::from_fn` with a
proper `tower::Layer` implementation for maximum composability. This
is a Rust learning exercise as much as a functional need - it
teaches generics, associated types, and `Pin<Box<dyn Future>>`.
