# Design: wicket v0

Domain-gated static file server. Serves static files behind Google
OAuth, restricted to a single Google Workspace domain. Designed for
deployment on Google Cloud Run.

## Architecture

Wicket's core is an Axum auth middleware layer. Everything else is
pluggable around it.

**Auth layer (the library):** a reusable `tower::Layer` that handles
Google OAuth, session cookies, and domain gating. It can wrap any Axum
service. This is the thing other projects can depend on.

**Static file backend (one possible inner service):** serves files
from an embedded directory or a local path. This sits behind the auth
layer. It could be swapped for a reverse proxy, an API, or anything
else that speaks HTTP.

**The binary (thin wiring):** reads config from environment variables,
picks a backend, wraps it with the auth layer, and starts the server.

This means wicket is a single crate that exposes both a library and a
binary. The library owns auth. The binary owns the specific use case
of serving static files behind that auth.

The same architecture supports the future path+method access rules -
the rule engine replaces the simple domain check inside the auth
layer, and nothing downstream changes.

## Request flow

1. Client requests any path.
2. Auth middleware checks for a valid session cookie.
3. No valid session: redirect to `/auth/login`, which starts the
   Google OAuth flow.
4. User authenticates with Google, Google redirects back to
   `/auth/callback` with an authorization code.
5. Wicket exchanges the code for tokens, verifies the ID token JWT
   (signature via Google's JWKS, issuer, audience, expiry), and checks
   the `hd` claim matches the configured domain.
6. On success, wicket sets a signed session cookie and redirects the
   user to their original destination.
7. Subsequent requests: middleware verifies the cookie signature and
   expiry, then passes the request through to the inner service.

## Reserved routes

- `GET /auth/login` - start OAuth flow
- `GET /auth/callback` - OAuth callback
- `GET /auth/logout` - clear session, redirect to `/`

All other routes are forwarded to the inner service (static files in
the default binary).

## OAuth

Standard Authorization Code flow with Google's OAuth 2.0 endpoints.
The `hd` parameter in the authorization URL hints Google to show only
accounts from the configured domain. The `hd` claim in the returned
ID token is verified server-side - this is the actual enforcement.

Additional layer: if the GCP project belongs to the same Workspace
org, setting the OAuth consent screen to "Internal" blocks non-domain
users at Google's end before they even get a token.

CSRF protection: a random `state` parameter is stored in a short-lived
cookie before the redirect to Google, and verified on callback.

## Sessions

Stateless signed cookies. No server-side session store.

The cookie contains the user's email, domain, and expiry. It is signed
with HMAC-SHA256 using a secret from config. It is `HttpOnly`,
`Secure`, `SameSite=Lax`.

Session duration is configurable (default 8 hours). No refresh token
flow for v0 - expired sessions go through the OAuth flow again, which
is typically skip-through for returning Google users.

## JWKS caching

Google's public signing keys are fetched from their JWKS endpoint and
cached in memory. Cache lifetime follows the `Cache-Control` header
from Google's response. Keys are re-fetched on expiry or when token
verification encounters an unknown key ID (handles key rotation).

## Static file serving

Two modes:

- **Embedded** (production): static files are baked into the binary at
  compile time. Single binary deployment, no external dependencies.
- **Directory** (development): an environment variable points to a
  local directory. Files are served from disk. Avoids recompilation on
  every content change.

SPA fallback (serve `index.html` for unmatched paths) is on by default.

## Configuration

All configuration via environment variables:

- `WICKET_CLIENT_ID` - Google OAuth client ID (required)
- `WICKET_CLIENT_SECRET` - Google OAuth client secret (required)
- `WICKET_DOMAIN` - allowed Workspace domain (required)
- `WICKET_SESSION_SECRET` - HMAC signing key, base64, >= 32 bytes (required)
- `WICKET_SESSION_TTL` - session duration, e.g. `8h`, `30m` (default `8h`)
- `WICKET_LISTEN_ADDR` - bind address (default `0.0.0.0:8080`)
- `WICKET_STATIC_DIR` - local directory override for dev mode

## Deployment

Multi-stage Docker build: compile with the official Rust image, copy
the binary into a minimal distroless runtime image. Static files are
embedded, so the runtime image contains only the binary.

Cloud Run provides custom domain support with managed TLS, horizontal
scaling, and scale-to-zero.

## Build tooling

Uses [just](https://github.com/casey/just) for build recipes. The
justfile will be introduced when there is something to build, not
before.

## Build order

Each step is independently testable and committable.

1. **Skeleton** - `cargo init`, dependencies, Axum hello-world.
   Verify it compiles and serves on localhost.
2. **Static file serving** - embedded files and directory override.
   Sample `index.html`.
3. **Config** - environment variable parsing into a config struct,
   passed as Axum state.
4. **Session** - cookie signing and verification.
5. **Auth middleware** - the layer that checks sessions and redirects.
   At this point the site is locked - every request redirects.
6. **OAuth flow** - login, callback, logout routes. JWKS fetching.
   This unlocks the site for domain users.
7. **Dockerfile and Cloud Run** - containerize, deploy, verify
   end-to-end.

## Dependencies

Core: `axum`, `tokio`, `tower-http`, `reqwest`, `jsonwebtoken`,
`serde`, `rust-embed`, `cookie`, `hmac`, `sha2`, `base64`, `rand`.

Logging: `tracing`, `tracing-subscriber`.

## Open questions

- **HTTPS in dev**: session cookies are `Secure`, which doesn't work
  over plain HTTP. Options: detect localhost and skip the Secure flag,
  or set up a self-signed cert for local dev.
- **Error UX**: what does the user see when domain check fails? A
  minimal "access denied" page with a link to retry with a different
  account.
- **SPA fallback**: should it be on by default, or opt-in via config?
