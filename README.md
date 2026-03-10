# golang-gin-best-practices

Agent Skills for building production-grade REST APIs with Go and the Gin framework.

[![Release](https://img.shields.io/github/v/release/henriqueatila/golang-gin-best-practices?label=release)](https://github.com/henriqueatila/golang-gin-best-practices/releases/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Skills.sh](https://img.shields.io/badge/skills.sh-golang--gin--best--practices-blue?logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCIgZmlsbD0id2hpdGUiPjxwYXRoIGQ9Ik0xMiAyTDIgNy41djlMMTIgMjJsMTAtNS41di05TDEyIDJ6Ii8+PC9zdmc+)](https://skills.sh/henriqueatila/golang-gin-best-practices)

---

## Skills

| Skill | Description | Install |
|---|---|---|
| **golang-gin-architect** | Software architect — system design, complexity assessment, API design, skill orchestration | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-architect` |
| **golang-gin-api** | Core REST API — routing, handlers, binding, error handling, project structure | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-api` |
| **golang-gin-auth** | JWT authentication, login handler, RBAC middleware, token lifecycle | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-auth` |
| **golang-gin-database** | PostgreSQL with GORM or sqlx, repository pattern, migrations, connection pooling | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-database` |
| **golang-gin-psql-dba** | PostgreSQL DBA — schema design, index strategy, migration safety, extensions (ParadeDB, pgvector, PostGIS, TimescaleDB) | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-psql-dba` |
| **golang-gin-testing** | Unit tests with httptest, integration tests with testcontainers, e2e flows | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-testing` |
| **golang-gin-deploy** | Multi-stage Dockerfile, docker-compose, Kubernetes manifests, CI/CD pipelines | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-deploy` |

---

## Quick Start

### Install all skills

```bash
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-architect
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-api
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-auth
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-database
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-psql-dba
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-testing
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-deploy
```

### Install individual skills

```bash
# Architecture brain — system design, complexity assessment, skill orchestration
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-architect

# Core API only
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-api

# Add authentication to an existing project
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-auth

# Add database layer
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-database

# Add DBA/architect guidance (schema design, indexes, extensions)
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-psql-dba

# Add testing infrastructure
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-testing

# Add deployment configuration
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-deploy
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

- **golang-gin-architect** is the "brain" — assesses complexity, selects patterns, routes to specific skills, and orchestrates workflows
- **golang-gin-api** defines the project structure, `AppError` type, and handler patterns that all other skills follow
- **golang-gin-auth** provides JWT middleware consumed by **golang-gin-api** route groups
- **golang-gin-database** provides the `UserRepository` interface and implementations used by **golang-gin-auth** and **golang-gin-api** handlers
- **golang-gin-psql-dba** provides PostgreSQL architecture decisions (schema design, indexes, migrations, extensions) that inform **golang-gin-database** implementations
- **golang-gin-testing** tests the handlers, services, and middleware defined across all skills
- **golang-gin-deploy** builds the project structure from **golang-gin-api** and wires health checks from **golang-gin-api**

Start with `golang-gin-architect` for architecture decisions, `golang-gin-api` for implementation, and add skills incrementally as your project grows.

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
│   ├── golang-gin-architect/
│   │   ├── SKILL.md                 # System design, complexity assessment, skill orchestration
│   │   ├── metadata.json
│   │   └── references/
│   │       ├── complexity-assessment.md   # Decision trees, complexity budget, pattern gates
│   │       ├── system-design.md           # C4 model, bounded contexts, domain modeling
│   │       ├── data-patterns.md           # CQRS, event sourcing, saga, transactional outbox
│   │       ├── resilience-patterns.md     # Circuit breaker, bulkhead, retry, rate limiting
│   │       ├── api-design.md             # Versioning, pagination, backwards compatibility
│   │       ├── cross-cutting-concerns.md  # Observability, caching, security, feature flags
│   │       ├── adr-templates.md           # Architecture Decision Record templates
│   │       ├── skill-orchestration.md     # When to activate each gingo skill
│   │       ├── tech-debt-management.md    # Debt quadrant, prioritization, communication
│   │       ├── clean-architecture.md     # Layers → Go packages, ports & adapters, DI
│   │       ├── redis-caching-strategy.md # Smart caching, stampede prevention, sessions
│   │       ├── messaging-patterns.md     # RabbitMQ producer/consumer, DLQ, pub/sub
│   │       ├── object-storage.md         # S3/MinIO upload, presigned URLs, multipart
│   │       ├── error-flow-architecture.md # Error flow domain→service→handler
│   │       ├── golden-main-template.md   # Production-ready main.go templates
│   │       ├── grpc-interop.md           # Gin HTTP + gRPC coexistence
│   │       └── data-ownership.md         # Database-per-service, data sync
│   │
│   ├── golang-gin-api/
│   │   ├── SKILL.md                 # Server setup, routing, handlers, binding, errors
│   │   ├── metadata.json
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
│   │   └── references/
│   │       ├── jwt-patterns.md      # Token refresh, blacklisting (Redis), RS256 vs HS256
│   │       └── rbac.md              # RequireRole, permissions, multi-tenant authorization
│   │
│   ├── golang-gin-database/
│   │   ├── SKILL.md                 # Repository pattern, GORM/sqlx, connection pooling, DI
│   │   ├── metadata.json
│   │   └── references/
│   │       ├── gorm-patterns.md     # Models, CRUD, soft deletes, transactions, hooks
│   │       ├── sqlx-patterns.md     # Struct scanning, NamedExec, IN clauses, transactions
│   │       └── migrations.md        # golang-migrate CLI, zero-downtime, seeding, rollback
│   │
│   ├── golang-gin-psql-dba/
│   │   ├── SKILL.md                 # Schema design, index strategy, migration safety, extensions
│   │   ├── metadata.json
│   │   └── references/
│   │       ├── schema-design.md             # Naming, types, constraints, multi-tenancy, audit
│   │       ├── migration-impact-analysis.md # Lock levels, zero-downtime ALTER TABLE patterns
│   │       ├── index-strategy.md            # B-tree, GIN, GiST, BRIN, EXPLAIN ANALYZE
│   │       ├── query-performance.md         # pg_stat_statements, autovacuum, pool sizing
│   │       ├── extensions-toolkit.md        # pg_cron, pg_partman, pg_trgm, pgcrypto
│   │       ├── paradedb-full-text-search.md # BM25 search, hybrid search, analytics
│   │       ├── pgvector-embeddings.md       # Vector storage, HNSW/IVFFlat, similarity
│   │       ├── postgis-geospatial.md        # Spatial types, distance queries, GiST indexes
│   │       ├── timescaledb-time-series.md   # Hypertables, compression, retention
│   │       ├── row-level-security.md        # RLS policies, multi-tenant isolation
│   │       ├── backup-and-recovery.md       # pg_dump, WAL archiving, PITR
│   │       └── replication-and-ha.md        # Streaming replication, Patroni, failover
│   │
│   ├── golang-gin-testing/
│   │   ├── SKILL.md                 # httptest, table-driven tests, mock repositories
│   │   ├── metadata.json
│   │   └── references/
│   │       ├── unit-tests.md        # Handler tests, middleware isolation, mock generation
│   │       ├── integration-tests.md # testcontainers, TestMain, DB lifecycle, cleanup
│   │       └── e2e.md               # Full flows, docker-compose tests, GitHub Actions CI
│   │
│   ├── golang-gin-deploy/
│   │   ├── SKILL.md                 # Multi-stage Dockerfile, docker-compose, health checks
│   │   ├── metadata.json
│   │   └── references/
│   │       ├── dockerfile.md        # Distroless, build args, layer caching, image size
│   │       ├── docker-compose.md    # Air hot reload, pgadmin, networking, integration tests
│   │       ├── kubernetes.md        # Deployment, Service, ConfigMap, HPA, Ingress, Helm
│   │       └── observability.md     # OpenTelemetry tracing, metrics, slog correlation
│   │
│   ├── golang-gin-api.zip                  # Packaged skill (SKILL.md + references/)
│   ├── golang-gin-auth.zip
│   ├── golang-gin-database.zip
│   ├── golang-gin-deploy.zip
│   └── golang-gin-testing.zip
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
