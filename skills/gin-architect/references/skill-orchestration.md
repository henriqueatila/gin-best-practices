# Skill Orchestration Guide

How and when to activate each skill in the gingo ecosystem. Decision matrix, common workflows, and skill composition patterns.

## Table of Contents

- [Skill Overview](#skill-overview)
- [Decision Matrix](#decision-matrix)
- [Common Workflows](#common-workflows)
- [Skill Composition Patterns](#skill-composition-patterns)
- [Skill Boundary Rules](#skill-boundary-rules)
- [Troubleshooting: Which Skill?](#troubleshooting-which-skill)

---

## Skill Overview

| Skill | Domain | One-line Purpose |
|---|---|---|
| **gin-architect** | Architecture | System design, complexity assessment, pattern selection, skill routing |
| **gin-api** | HTTP Layer | Routing, handlers, binding, middleware, error handling |
| **gin-auth** | Security | JWT, RBAC, login/register, token lifecycle, protected routes |
| **gin-database** | Data Access | GORM/sqlx wiring, repository pattern, migrations tooling, transactions |
| **gin-psql-dba** | Database | Schema design, index strategy, query optimization, extensions, migration safety |
| **gin-testing** | Quality | Unit/integration/e2e tests, httptest, testcontainers, table-driven tests |
| **gin-deploy** | Operations | Docker, docker-compose, Kubernetes, CI/CD, 12-factor config |

**Key distinction:**
- `gin-database` = **how to write Go code** that talks to PostgreSQL (GORM, sqlx, repository pattern)
- `gin-psql-dba` = **how to design and optimize PostgreSQL** (schemas, indexes, migrations, extensions)
- `gin-architect` = **when to use what** and how skills compose together

---

## Decision Matrix

Use this to determine which skill(s) to activate for a given task.

### By Task Type

| Task | Primary Skill | May Also Need |
|---|---|---|
| "Create a new endpoint" | gin-api | gin-database, gin-testing |
| "Add login/signup" | gin-auth | gin-api, gin-database |
| "Design the schema" | gin-psql-dba | gin-database (migration tooling) |
| "Write repository code" | gin-database | gin-psql-dba (schema decisions) |
| "Query is slow" | gin-psql-dba | gin-database (query code) |
| "Add tests" | gin-testing | (reads other skills for patterns) |
| "Dockerize the app" | gin-deploy | — |
| "Set up CI/CD" | gin-deploy | gin-testing (test step) |
| "Should I use microservices?" | gin-architect | — |
| "How to structure this feature?" | gin-architect | gin-api, gin-database |
| "Add caching" | gin-architect | gin-api (middleware) |
| "Add full-text search" | gin-psql-dba | gin-database (Go code) |
| "Add WebSocket support" | gin-api | — |
| "Rate limit endpoints" | gin-api | — |
| "Set up monitoring" | gin-deploy | gin-architect (observability design) |

### By Keyword

| If the user mentions... | Activate |
|---|---|
| route, handler, middleware, binding, JSON response | gin-api |
| JWT, token, login, signup, RBAC, permission, role | gin-auth |
| GORM, sqlx, repository, migration tool, transaction | gin-database |
| schema, index, EXPLAIN, ALTER TABLE, extension, pgvector, PostGIS | gin-psql-dba |
| test, httptest, testcontainers, mock, coverage | gin-testing |
| Docker, Kubernetes, CI/CD, deploy, helm, health check | gin-deploy |
| architecture, microservice, monolith, CQRS, pattern, scale, design | gin-architect |

---

## Common Workflows

### New Feature (End-to-End)

```
1. gin-architect  → Assess complexity, choose patterns
2. gin-psql-dba   → Design schema, plan migration
3. gin-database   → Implement repository + migration files
4. gin-api        → Implement handler + service + routes
5. gin-auth       → Add auth middleware (if protected)
6. gin-testing    → Write unit + integration tests
7. gin-deploy     → Update Docker/K8s configs (if needed)
```

**For simple CRUD features (80% of cases), skip step 1** — go directly to schema → repository → handler → tests.

### Add Authentication to Existing App

```
1. gin-auth       → JWT middleware, login/register handlers, RBAC
2. gin-database   → User model, token store (if refresh tokens in DB)
3. gin-psql-dba   → User table schema, indexes on email
4. gin-api        → Wire auth middleware to route groups
5. gin-testing    → Auth test helpers, protected route tests
```

### Performance Investigation

```
1. gin-psql-dba   → EXPLAIN ANALYZE, index analysis, query optimization
2. gin-architect  → Caching strategy, read replicas decision
3. gin-database   → Optimize repository queries, add connection pool tuning
4. gin-testing    → Benchmark tests
5. gin-deploy     → Horizontal scaling, resource limits
```

### Database Migration (Schema Change)

```
1. gin-psql-dba   → Migration safety analysis (lock levels, zero-downtime)
2. gin-database   → Write migration files (golang-migrate)
3. gin-testing    → Test migration up/down
4. gin-deploy     → Update deployment to run migration before app starts
```

### Greenfield Project Setup

```
1. gin-architect  → Project structure decision, technology choices, ADRs
2. gin-api        → Server setup, project layout, base middleware
3. gin-database   → Database connection, initial schema
4. gin-psql-dba   → Schema design, initial indexes
5. gin-auth       → Auth setup (if needed from start)
6. gin-testing    → Test infrastructure, CI integration
7. gin-deploy     → Docker, docker-compose for local dev
```

---

## Skill Composition Patterns

### Pattern 1: Schema-First (Recommended)

Design the database first, then work upward.

```
gin-psql-dba (schema) → gin-database (repository) → gin-api (handler) → gin-testing
```

**Why:** Schema changes are the hardest to modify later. Get the data model right first.

### Pattern 2: API-First

Design the API contract first, then work downward.

```
gin-api (contract/types) → gin-database (repository to match) → gin-psql-dba (schema to match)
```

**When:** External API consumers need to see the contract before implementation. Use OpenAPI spec as the starting point.

### Pattern 3: Test-First (TDD)

Write tests first, then implement.

```
gin-testing (test) → gin-api (handler) → gin-database (repository) → gin-psql-dba (schema)
```

**When:** Requirements are very clear and well-defined. Each test drives the next implementation layer.

---

## Skill Boundary Rules

### What Each Skill Should NOT Do

| Skill | Should NOT |
|---|---|
| gin-api | Write SQL queries, design schemas, make auth decisions |
| gin-auth | Design database schemas, set up deployment |
| gin-database | Decide index types, analyze EXPLAIN plans, choose extensions |
| gin-psql-dba | Write Go repository code, implement handlers |
| gin-testing | Implement features, change production code |
| gin-deploy | Make architecture decisions, change business logic |
| gin-architect | Write implementation code (routes to specific skills instead) |

### Overlap Resolution

| Overlap Area | Owner | Other Skill's Role |
|---|---|---|
| Database connection setup | gin-database | gin-psql-dba advises pool sizing |
| Migration files | gin-database | gin-psql-dba reviews safety |
| Auth middleware | gin-auth | gin-api wires it to routes |
| Error handling | gin-api | gin-architect defines strategy |
| Docker health checks | gin-deploy | gin-api implements `/health` endpoint |
| Test infrastructure | gin-testing | gin-deploy sets up test DB in CI |

---

## Troubleshooting: Which Skill?

### "I'm not sure which skill to use"

Ask these questions:

1. **Is this about code or decisions?**
   - Code → specific skill (gin-api, gin-database, gin-auth, etc.)
   - Decisions → gin-architect

2. **Is this about HTTP or data?**
   - HTTP (routes, middleware, handlers) → gin-api
   - Data (queries, schema, migrations) → gin-database or gin-psql-dba

3. **Is this about Go code or PostgreSQL?**
   - Go code that talks to DB → gin-database
   - PostgreSQL itself (DDL, indexes, EXPLAIN) → gin-psql-dba

4. **Is this about running the app or building it?**
   - Building → gin-api, gin-database, gin-auth
   - Running → gin-deploy

5. **Is this about verifying correctness?**
   - Yes → gin-testing

### Common Misroutes

| User Says | Seems Like | Actually |
|---|---|---|
| "Add a database" | gin-database | gin-psql-dba (schema) THEN gin-database (code) |
| "Make it faster" | gin-architect | gin-psql-dba first (90% of perf issues are DB) |
| "Add middleware" | gin-api | gin-auth if it's auth middleware |
| "Set up migrations" | gin-psql-dba | gin-database (tooling) with gin-psql-dba (safety review) |
| "Scale the app" | gin-deploy | gin-architect first (is scaling the right answer?) |
