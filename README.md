# golang-gin-best-practices

Agent Skills for building production-grade REST APIs with Go and the Gin framework.

[![Release](https://img.shields.io/github/v/release/henriqueatila/golang-gin-best-practices?label=release)](https://github.com/henriqueatila/golang-gin-best-practices/releases/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Skills.sh](https://img.shields.io/badge/skills.sh-golang--gin--best--practices-blue?logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCIgZmlsbD0id2hpdGUiPjxwYXRoIGQ9Ik0xMiAyTDIgNy41djlMMTIgMjJsMTAtNS41di05TDEyIDJ6Ii8+PC9zdmc+)](https://skills.sh/henriqueatila/golang-gin-best-practices)

---

## Skills

| Skill | Description | Install |
|---|---|---|
| **golang-gin-api** | Core REST API — routing, handlers, binding, error handling, security headers, CORS, project structure | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-api` |
| **golang-gin-auth** | JWT auth, RBAC middleware, token lifecycle, CSRF protection, rate limiting, bcrypt, secure cookies | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-auth` |
| **golang-gin-database** | PostgreSQL with GORM/sqlx, repository pattern, migrations, connection retry, cursor pagination, transactions | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-database` |
| **golang-gin-testing** | Unit/integration/e2e tests, benchmarks, fuzz testing, golden files, testcontainers, coverage thresholds | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-testing` |
| **golang-gin-deploy** | Multi-stage Docker, K8s (PDB, NetworkPolicy, HPA), Trivy scanning, CI/CD, observability | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-deploy` |
| **golang-gin-swagger** | Swagger/OpenAPI with swaggo/swag, full CRUD + auth annotations, Swagger UI, CI/CD | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-swagger` |

---

## Quick Start

### Install all skills at once

```bash
npx skills add henriqueatila/golang-gin-best-practices
```

### Install individual skills

```bash
# Core API only
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-api

# Add authentication to an existing project
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-auth

# Add database layer
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-database

# Add testing infrastructure
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-testing

# Add deployment configuration
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-deploy

# Add Swagger/OpenAPI documentation
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-swagger
```

### Manual installation

Download the zip package for any skill from the `skills/` directory and extract it into your project's `.claude/skills/` folder:

```bash
# Example: download and extract golang-gin-api
curl -L https://github.com/henriqueatila/golang-gin-best-practices/raw/main/skills/golang-gin-api.zip -o golang-gin-api.zip
unzip golang-gin-api.zip -d .claude/skills/golang-gin-api/
```

---

## Architecture

### Progressive Disclosure

Each skill follows a two-level structure:

1. **SKILL.md** — The 80% you need daily. Loaded automatically by the agent. Covers the most common patterns with complete, compilable code examples. Under 500 lines.
2. **references/*.md** — The 20% for specific scenarios. Loaded on demand when you need deeper detail. Each file focuses on one topic.

This means the agent loads minimal context by default and fetches reference files only when needed — keeping token usage low while providing comprehensive coverage.

### Composability

Skills are designed to work together with shared conventions:

- **golang-gin-api** defines the project structure, `AppError` type, and handler patterns that all other skills follow
- **golang-gin-auth** provides JWT middleware consumed by **golang-gin-api** route groups
- **golang-gin-database** provides the `UserRepository` interface and implementations used by **golang-gin-auth** and **golang-gin-api** handlers
- **golang-gin-testing** tests the handlers, services, and middleware defined across all skills
- **golang-gin-deploy** builds the project structure from **golang-gin-api** and wires health checks from **golang-gin-api**
- **golang-gin-swagger** documents the handlers from **golang-gin-api**, the JWT security from **golang-gin-auth**, and integrates with **golang-gin-deploy** Docker builds

You can start with `golang-gin-api` and add skills incrementally as your project grows.

---

## Directory Structure

```
golang-gin-best-practices/
├── README.md
├── CLAUDE.md                        # Agent guidance for this repo
├── AGENTS.md                        # Multi-agent guidance
├── LICENSE
│
├── skills/
│   ├── golang-gin-api/
│   │   ├── SKILL.md                 # Server setup, routing, handlers, binding, errors
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── routing.md           # Route groups, versioning, pagination, file uploads
│   │       ├── middleware.md        # CORS, rate limiting, request ID, timeout, recovery
│   │       ├── error-handling.md    # AppError system, validation errors, panic recovery
│   │       ├── websocket.md         # gorilla/websocket, hub pattern, auth, ping/pong
│   │       └── rate-limiting.md     # Token bucket, sliding window, Redis, tiered limits
│   │
│   ├── golang-gin-auth/
│   │   ├── SKILL.md                 # JWT middleware, login handler, RBAC, token lifecycle
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── jwt-patterns.md      # Token refresh, blacklisting (Redis), RS256 vs HS256
│   │       └── rbac.md              # RequireRole, permissions, multi-tenant authorization
│   │
│   ├── golang-gin-database/
│   │   ├── SKILL.md                 # Repository pattern, GORM/sqlx, connection pooling, DI
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── gorm-patterns.md     # Models, CRUD, soft deletes, transactions, hooks
│   │       ├── sqlx-patterns.md     # Struct scanning, NamedExec, IN clauses, transactions
│   │       └── migrations.md        # golang-migrate CLI, zero-downtime, seeding, rollback
│   │
│   ├── golang-gin-testing/
│   │   ├── SKILL.md                 # httptest, table-driven tests, mock repositories
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── unit-tests.md        # Handler tests, middleware isolation, mock generation
│   │       ├── integration-tests.md # testcontainers, TestMain, DB lifecycle, cleanup
│   │       └── e2e.md               # Full flows, docker-compose tests, GitHub Actions CI
│   │
│   ├── golang-gin-deploy/
│   │   ├── SKILL.md                 # Multi-stage Dockerfile, docker-compose, health checks
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── dockerfile.md        # Distroless, build args, layer caching, image size
│   │       ├── docker-compose.md    # Air hot reload, pgadmin, networking, integration tests
│   │       ├── kubernetes.md        # Deployment, Service, ConfigMap, HPA, Ingress, Helm
│   │       └── observability.md     # OpenTelemetry tracing, metrics, slog correlation
│   │
│   ├── golang-gin-swagger/
│   │   ├── SKILL.md                 # Swagger UI, annotations, model tags, doc generation
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── annotations.md       # Complete annotation ref: @Param, uploads, enums, auth
│   │       └── ci-cd.md             # GitHub Actions, PR validation, OpenAPI 3.0, Docker
│   │
│   ├── golang-gin-api.zip           # Packaged skill (SKILL.md + references/)
│   ├── golang-gin-auth.zip
│   ├── golang-gin-database.zip
│   ├── golang-gin-deploy.zip
│   ├── golang-gin-testing.zip
│   └── golang-gin-swagger.zip
```

---

## Design Principles

**Opinionated patterns** — Each skill teaches one right way, not five options. The patterns are chosen for production correctness, not tutorial simplicity.

**Enterprise-audited** — Every skill passed a multi-agent enterprise audit covering correctness, security, completeness, and production-readiness. All code examples compile, handle errors properly, and follow OWASP security guidelines.

**PostgreSQL default** — All database examples target PostgreSQL. GORM and sqlx are both covered; the repository interface allows swapping implementations without touching business logic.

**Production-first** — Examples use `gin.New()` + explicit middleware instead of `gin.Default()`, `log/slog` instead of `fmt.Println`, `ShouldBind*` instead of `Bind*`, and environment variables instead of hardcoded configuration.

**Go 1.24+** — All code targets Go 1.24 or later. Uses standard library features (`log/slog`, `context`, `errors.As`) and avoids deprecated patterns.

**Thin handlers** — Handlers bind input, call a service, and format the response. Business logic lives in services. Data access lives in repositories. This is enforced consistently across all skills.

**Context propagation** — All blocking calls receive `c.Request.Context()` (handlers) or a propagated `context.Context` (services and repositories). This enables proper cancellation and tracing.

---

## Security

All skills enforce enterprise security patterns:

- **OWASP headers** — `X-Content-Type-Options`, `X-Frame-Options`, `CSP`, `HSTS`, `Permissions-Policy`
- **JWT** — `jti` for token blacklisting, `Issuer`/`Audience` validation, `NotBefore` claims
- **Auth hardening** — bcrypt (cost 12), rate limiting on auth endpoints, CSRF double-submit cookies, secure cookie flags
- **Database** — parameterized queries only, `pgconn.PgError` typed assertions, TLS/sslmode guidance
- **Containers** — non-root, read-only rootfs, dropped capabilities, seccomp, Trivy image scanning
- **No internal error leakage** — all error responses use generic messages; real errors logged server-side via `slog`

---

## Contributing

1. Follow the design principles above — new patterns must be production-ready
2. Code examples must compile and handle errors (no `_` for errors, no `fmt.Println`)
3. SKILL.md files must stay under 500 lines — move detail to reference files
4. Reference files over 300 lines must include a table of contents
5. Use `gin.New()`, `log/slog`, `ShouldBind*`, and `context.Context` consistently
6. Verify all Gin API calls match official documentation before submitting a PR

---

## License

MIT License — see [LICENSE](LICENSE) for details.
