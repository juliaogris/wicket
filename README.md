# wicket

A small gate for your web content.

Wicket is an HTTP server written in Rust that serves static files behind
Google OAuth, restricting access to users in a given Google Workspace
domain. Designed for deployment on Google Cloud Run.

## Goals

**v0: domain-gated static site**

- Serve static files from an embedded directory
- Authenticate users via Google OAuth 2.0
- Restrict access to a configured Google Workspace domain (via the `hd`
  claim in the ID token)
- Manage sessions with signed cookies

**Future: path and method access rules**

Extend wicket into an identity-aware HTTP gateway with per-path access
rules, inspired by Firebase security rules. First match wins, unmatched
requests are denied.

```yaml
rules:
  - path: /admin/**
    if: 'contains(user.roles, "admin")'

  - path: /users/{USER_ID}/**
    methods: [GET, PUT]
    if: 'user.name == USER_ID'

  - path: /public/**
    methods: [GET]
```

Path variables like `{USER_ID}` are captured and available in identity
expressions. This turns wicket from a simple auth gate into a reviewable,
self-contained access policy layer.

## Stack

- [Axum](https://github.com/tokio-rs/axum) - HTTP framework
- Google OAuth 2.0 for authentication
- Google Cloud Run for deployment

## Development

Requires [just](https://github.com/casey/just) for task running when
there is something to build.

## License

MIT
