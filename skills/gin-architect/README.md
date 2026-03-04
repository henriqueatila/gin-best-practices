# gin-architect

Pragmatic software architecture for Go Gin APIs. The "brain" that orchestrates all other gin skills.

## What's Included

- **SKILL.md** — Complexity budget, architecture decision trees, project structure by scale, API design rules, skill orchestration matrix, cross-cutting concerns, ADR template, tech debt assessment
- **references/complexity-assessment.md** — Full decision trees, complexity budget framework, right-size thinking, pattern selection matrix, "you don't need this yet" gates
- **references/system-design.md** — C4 model, bounded contexts, domain modeling, dependency graphs, Go package layout at scale
- **references/data-patterns.md** — CQRS, event sourcing, saga, transactional outbox, read replicas — high-complexity patterns with prerequisite gates and Go examples
- **references/resilience-patterns.md** — Circuit breaker, bulkhead, retry with backoff, rate limiting — low-cost resilience patterns for external dependencies
- **references/api-design.md** — Versioning, cursor/offset pagination, filtering, bulk operations, deprecation, backwards compatibility
- **references/cross-cutting-concerns.md** — Observability (slog, Prometheus, OpenTelemetry), caching (in-memory, Redis, HTTP), security architecture, feature flags, configuration
- **references/adr-templates.md** — ADR format, templates for database choice, auth strategy, caching, service extraction, worked examples
- **references/skill-orchestration.md** — When to activate each gingo skill, common workflows, composition patterns, boundary rules
- **references/tech-debt-management.md** — Debt quadrant, identification signals, measurement, prioritization matrix, refactoring strategies, stakeholder communication
- **references/clean-architecture.md** — Uncle Bob's layers → Go packages, dependency rule, ports & adapters, manual DI, feature module example, common mistakes
- **references/redis-caching-strategy.md** — Smart caching: decision matrix, stampede prevention, warming, pub/sub invalidation, sessions, distributed locks
- **references/messaging-patterns.md** — RabbitMQ: producer/consumer, work queues, pub/sub, dead letter queues, idempotent consumers
- **references/object-storage.md** — S3/MinIO/R2: upload/download, presigned URLs, multipart, docker-compose dev setup
- **references/error-flow-architecture.md** — Error flow domain→service→handler, wrapping conventions, sentinel vs custom types, complete chain example
- **references/golden-main-template.md** — Production-ready main.go templates, startup sequence, graceful shutdown, DI wiring
- **references/grpc-interop.md** — Gin HTTP + gRPC coexistence, shared service layer, cmux, buf, gRPC-Gateway
- **references/data-ownership.md** — Database-per-service, API composition, data sync, migration from monolith

## Categories

`go` `gin` `architecture` `system-design` `patterns` `api-design` `tech-debt` `decision-making`

## Install

```bash
npx skills add henriqueatila/golang-gin-best-practices --skill gin-architect
```
