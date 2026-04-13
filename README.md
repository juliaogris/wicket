# wicket

A small gate for your web content.

Wicket is an HTTP server written in Rust that serves static files behind
Google OAuth, restricting access to users from configured Google
Workspace domains or individual email addresses. Designed for
deployment on Google Cloud Run.

## Goals

**v0: domain-gated static site**

- Serve static files from a local directory behind Google OAuth 2.0
- Restrict access by Google Workspace domain (`hd` claim) or
  individual email address
- Manage sessions with signed cookies
- Deploy on Cloud Run

**Future**

- YAML config file with multi-provider allow-lists (Google, GitHub)
- GitHub OAuth as a second identity provider
- Embedded static files for single-binary deployment
- Path and method access rules, inspired by Firebase security rules
- Compose with [hearth](https://github.com/juliaogris/hearth) as an
  auth layer for data APIs

```yaml
allow:
  google:
    domains:
      - goteleport.com
    emails:
      - julia.ogris@gmail.com
  github:
    orgs:
      - gravitational

rules:
  - path: /admin/**
    if: 'contains(user.roles, "admin")'
  - path: /data/user/{USER_ID}/**
    methods: [GET, PUT]
    if: 'user.name == USER_ID'
  - path: /public/**
    methods: [GET]
```

## Stack

- [Axum](https://github.com/tokio-rs/axum) - HTTP framework
- Google OAuth 2.0 for authentication
- Google Cloud Run for deployment

## Development

Requires [just](https://github.com/casey/just) for task running when
there is something to build.

## License

MIT
