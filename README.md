# gin-best-practices

Agent Skills for building production-grade REST APIs with Go and the Gin framework.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Skills

| Skill | Description | Install |
|---|---|---|
| **gin-api** | Core REST API — routing, handlers, binding, error handling, project structure | `npx skills add henriqueatila/gin-best-practices --skill gin-api` |
| **gin-auth** | JWT authentication, login handler, RBAC middleware, token lifecycle | `npx skills add henriqueatila/gin-best-practices --skill gin-auth` |
| **gin-database** | PostgreSQL with GORM or sqlx, repository pattern, migrations, connection pooling | `npx skills add henriqueatila/gin-best-practices --skill gin-database` |
| **gin-testing** | Unit tests with httptest, integration tests with testcontainers, e2e flows | `npx skills add henriqueatila/gin-best-practices --skill gin-testing` |
| **gin-deploy** | Multi-stage Dockerfile, docker-compose, Kubernetes manifests, CI/CD pipelines | `npx skills add henriqueatila/gin-best-practices --skill gin-deploy` |

---

## Quick Start

### Install all skills

```bash
npx skills add henriqueatila/gin-best-practices --skill gin-api
npx skills add henriqueatila/gin-best-practices --skill gin-auth
npx skills add henriqueatila/gin-best-practices --skill gin-database
npx skills add henriqueatila/gin-best-practices --skill gin-testing
npx skills add henriqueatila/gin-best-practices --skill gin-deploy
```

### Install individual skills

```bash
# Core API only
npx skills add henriqueatila/gin-best-practices --skill gin-api

# Add authentication to an existing project
npx skills add henriqueatila/gin-best-practices --skill gin-auth

# Add database layer
npx skills add henriqueatila/gin-best-practices --skill gin-database

# Add testing infrastructure
npx skills add henriqueatila/gin-best-practices --skill gin-testing

# Add deployment configuration
npx skills add henriqueatila/gin-best-practices --skill gin-deploy
```

### Manual installation

Download the zip package for any skill from the `skills/` directory and extract it into your project's `.claude/skills/` folder:

```bash
# Example: download and extract gin-api
curl -L https://github.com/henriqueatila/gin-best-practices/raw/main/skills/gin-api.zip -o gin-api.zip
unzip gin-api.zip -d .claude/skills/gin-api/
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

- **gin-api** defines the project structure, `AppError` type, and handler patterns that all other skills follow
- **gin-auth** provides JWT middleware consumed by **gin-api** route groups
- **gin-database** provides the `UserRepository` interface and implementations used by **gin-auth** and **gin-api** handlers
- **gin-testing** tests the handlers, services, and middleware defined across all skills
- **gin-deploy** builds the project structure from **gin-api** and wires health checks from **gin-api**

You can start with `gin-api` and add skills incrementally as your project grows.

---

## Directory Structure

```
gin-best-practices/
├── README.md
├── CLAUDE.md                        # Agent guidance for this repo
├── AGENTS.md                        # Multi-agent guidance
├── LICENSE
│
├── skills/
│   ├── gin-api/
│   │   ├── SKILL.md                 # Server setup, routing, handlers, binding, errors
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── routing.md           # Route groups, versioning, pagination, file uploads
│   │       ├── middleware.md        # CORS, rate limiting, request ID, timeout, recovery
│   │       └── error-handling.md    # AppError system, validation errors, panic recovery
│   │
│   ├── gin-auth/
│   │   ├── SKILL.md                 # JWT middleware, login handler, RBAC, token lifecycle
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── jwt-patterns.md      # Token refresh, blacklisting (Redis), RS256 vs HS256
│   │       └── rbac.md              # RequireRole, permissions, multi-tenant authorization
│   │
│   ├── gin-database/
│   │   ├── SKILL.md                 # Repository pattern, GORM/sqlx, connection pooling, DI
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── gorm-patterns.md     # Models, CRUD, soft deletes, transactions, hooks
│   │       ├── sqlx-patterns.md     # Struct scanning, NamedExec, IN clauses, transactions
│   │       └── migrations.md        # golang-migrate CLI, zero-downtime, seeding, rollback
│   │
│   ├── gin-testing/
│   │   ├── SKILL.md                 # httptest, table-driven tests, mock repositories
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── unit-tests.md        # Handler tests, middleware isolation, mock generation
│   │       ├── integration-tests.md # testcontainers, TestMain, DB lifecycle, cleanup
│   │       └── e2e.md               # Full flows, docker-compose tests, GitHub Actions CI
│   │
│   ├── gin-deploy/
│   │   ├── SKILL.md                 # Multi-stage Dockerfile, docker-compose, health checks
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── dockerfile.md        # Distroless, build args, layer caching, image size
│   │       ├── docker-compose.md    # Air hot reload, pgadmin, networking, integration tests
│   │       └── kubernetes.md        # Deployment, Service, ConfigMap, HPA, Ingress, Helm
│   │
│   ├── gin-api.zip                  # Packaged skill (SKILL.md + references/)
│   ├── gin-auth.zip
│   ├── gin-database.zip
│   ├── gin-deploy.zip
│   └── gin-testing.zip
```

---

## Design Principles

**Opinionated patterns** — Each skill teaches one right way, not five options. The patterns are chosen for production correctness, not tutorial simplicity.

**PostgreSQL default** — All database examples target PostgreSQL. GORM and sqlx are both covered; the repository interface allows swapping implementations without touching business logic.

**Production-first** — Examples use `gin.New()` + explicit middleware instead of `gin.Default()`, `log/slog` instead of `fmt.Println`, `ShouldBind*` instead of `Bind*`, and environment variables instead of hardcoded configuration.

**Go 1.24+** — All code targets Go 1.24 or later. Uses standard library features (`log/slog`, `context`, `errors.As`) and avoids deprecated patterns.

**Thin handlers** — Handlers bind input, call a service, and format the response. Business logic lives in services. Data access lives in repositories. This is enforced consistently across all skills.

**Context propagation** — All blocking calls receive `c.Request.Context()` (handlers) or a propagated `context.Context` (services and repositories). This enables proper cancellation and tracing.

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
