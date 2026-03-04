# Complexity Assessment Reference

This file is the deep-dive companion to `gin-architect/SKILL.md`. It covers the complexity budget framework, full decision trees, right-size thinking calibrated to team and product stage, pattern selection matrix, and "you don't need this yet" gates for every major architectural pattern. Load this when evaluating whether a proposed architecture is appropriately sized — or over-engineered.

## Table of Contents

1. [Complexity Budget Framework](#1-complexity-budget-framework)
2. [Decision Trees — Full Detail](#2-decision-trees--full-detail)
   - 2.1 [Monolith vs Microservices](#21-monolith-vs-microservices)
   - 2.2 [Sync vs Async](#22-sync-vs-async)
   - 2.3 [SQL vs NoSQL](#23-sql-vs-nosql)
   - 2.4 [DDD vs Simple CRUD](#24-ddd-vs-simple-crud)
   - 2.5 [Layered vs Clean vs Hexagonal Architecture](#25-layered-vs-clean-vs-hexagonal-architecture)
3. [Right-Size Thinking Matrix](#3-right-size-thinking-matrix)
4. [Pattern Selection Matrix](#4-pattern-selection-matrix)
5. [You Don't Need This Yet — Gates](#5-you-dont-need-this-yet--gates)

---

## 1. Complexity Budget Framework

Every architecture decision has a cost: cognitive load on the team, infrastructure to maintain, onboarding friction for new hires, and surface area for bugs. That cost is paid continuously — not just during initial implementation.

A **complexity budget** is the finite cognitive capacity a team can sustain before velocity degrades. When you exceed it, PRs slow down, bugs increase, and onboarding becomes painful. The goal is not to spend zero — it's to spend intentionally on things that pay off.

### Scoring Scale

Rate each architectural choice on a 1–5 complexity scale:

| Score | Meaning | Examples |
|---|---|---|
| 1 | Trivially simple — any Go dev knows it | Flat package layout, basic CRUD handlers |
| 2 | Low overhead — small learning curve | Repository pattern, service layer, Redis caching |
| 3 | Moderate overhead — requires planning and discipline | API Gateway, feature flags, modular monolith |
| 4 | High overhead — team needs dedicated ownership | CQRS, Saga/choreography, event-driven async |
| 5 | Expert territory — significant infra + operational cost | Event sourcing, service mesh, distributed tracing across 10+ services |

### Budget by Team and Stage

| Team Size | Product Stage | Max Recommended Score | Guidance |
|---|---|---|---|
| 1–3 devs | MVP / early | 10–12 | Flat layout + service + repository. Done. |
| 1–3 devs | Growth | 12–15 | Add feature modules and Redis if justified by data |
| 3–8 devs | MVP | 12–15 | Modular monolith. No microservices. |
| 3–8 devs | Growth | 15–20 | Extract 1–2 services max, only at proven pain points |
| 8–15 devs | Growth | 20–25 | Bounded modules, limited service extraction |
| 8–15 devs | Mature | 25–30 | Multiple services viable if infra maturity is there |
| 15+ devs | Mature | 30+ | Full distributed systems justified — but still measure ROI |

### Budget Example: 3-Dev Startup

A team of 3 proposes this stack for their MVP:

| Choice | Cost |
|---|---|
| Modular monolith | 3 |
| CQRS for orders domain | 4 |
| Event sourcing for audit trail | 5 |
| Kafka for async notifications | 4 |
| Service mesh (Istio) | 5 |
| **Total** | **21** |

Budget for a 3-dev MVP is 10–12. They blew it on the first sprint — before writing a single business rule.

**What they should build instead:**

| Choice | Cost |
|---|---|
| Flat handler/service/repository | 1 |
| PostgreSQL (transactions handle audit) | 1 |
| Goroutine + channel for email notifications | 1 |
| **Total** | **3** |

Three devs, three patterns, three points. Now they can move fast and add complexity only when pain demands it.

**What to do:** Before finalizing any architecture, list every non-trivial choice, score it, sum it up, compare to your budget. Justify anything that puts you over.

---

## 2. Decision Trees — Full Detail

### 2.1 Monolith vs Microservices

The default is a monolith. Extraction is a deliberate act with documented justification.

```
START: Are you starting a new project?
  └── Yes → MONOLITH. Come back to this tree when you feel real pain.

START: Existing system. Do you have measurable deploy coupling pain?
  (i.e., team A's deployment blocks team B's release regularly)
  ├── No → MODULAR MONOLITH. Clean package boundaries, shared DB, single deploy.
  └── Yes → Do you have 3+ teams that need to deploy independently?
      ├── No → MODULAR MONOLITH. Reorganize package ownership.
      └── Yes → Does the module have a clearly different scaling profile?
          (e.g., image processing at 10x the CPU of everything else)
          ├── No → Still MODULAR MONOLITH. Team autonomy != microservice.
          └── Yes → Do you have the infra maturity to run a service independently?
              (CI/CD per service, distributed tracing, service discovery, on-call)
              ├── No → BUILD INFRA MATURITY FIRST. Extract after.
              └── Yes → Extract THAT ONE MODULE into a service.
                        Keep everything else in the monolith.
                        Re-evaluate the next extraction in 6 months.
```

**Pain signals that justify extraction (all must be true):**
- Deploy coupling blocks releases more than once per month
- The module has independent scaling needs (measured, not hypothetical)
- The team boundary is stable and the API contract is well-defined
- You have observability in place (logs + traces + metrics across services)
- You have a rollback strategy if the service is unavailable

**Anti-patterns to call out:**

- **Resume-Driven Development**: "We should use microservices because it looks good on job postings." — Hard no.
- **Netflix Envy**: "Netflix uses microservices, so we should too." Netflix has 2,000+ engineers and spent years building the infra. You have 4.
- **Preemptive extraction**: "This module *might* need to scale independently someday." Someday is not a justification.
- **Micro-monolith**: Services that share a database are not microservices — they are a distributed monolith, which is strictly worse.

**The modular monolith sweet spot (90% of projects):**

```go
// internal/order/         ← team A owns this entire package
//   handler.go
//   service.go
//   repository.go
//   model.go
//
// internal/inventory/     ← team B owns this entire package
//   handler.go
//   service.go
//   repository.go
//   model.go
//
// Cross-module calls: order.Service calls inventory.Service via interface.
// No shared database queries across packages. No circular imports.
// Same binary. One deploy. Zero distributed systems complexity.
```

This gives you team autonomy, clear boundaries, and independent testability — without distributed systems overhead.

**What to do:** If you feel the urge to "start with microservices," start with a modular monolith instead. Design clean package boundaries. When you feel *real, measured* pain from the monolith — not theoretical pain — then extract exactly one service. Repeat.

---

### 2.2 Sync vs Async

The default is synchronous HTTP. Async has a coordination cost that must be earned.

```
START: Does the caller need the result to proceed?
  └── Yes → SYNCHRONOUS HTTP. Done. Don't read further.

START: Caller doesn't need the result immediately.
  └── Is the operation fast (< 200ms, no external I/O)?
      ├── Yes → Still SYNCHRONOUS. Async overhead isn't worth it.
      └── No → Is failure acceptable? (retry-later semantics are OK)
          ├── No → SYNCHRONOUS with timeout + retry + circuit breaker.
          │         Async won't fix unreliable dependencies.
          └── Yes → Is the work CPU-bound or I/O-bound?
              ├── CPU-bound (image resize, PDF gen, ML inference) →
              │   Goroutine pool + work channel (bounded concurrency).
              │   No external queue needed.
              └── I/O-bound (email, webhook, push notification) →
                  ├── Volume < 1K/min → Goroutine + channel. Simple.
                  └── Volume > 1K/min or needs persistence →
                      Message queue (Redis Streams, SQS, RabbitMQ).
                      └── Need exactly-once delivery?
                          ├── No → Simple queue with at-least-once.
                          └── Yes → Transactional outbox + idempotent consumers.
```

**Go examples by level:**

Level 1 — goroutine + channel (no queue, bounded concurrency):
```go
// internal/notification/worker.go
package notification

import (
    "context"
    "log/slog"
)

type EmailJob struct {
    To      string
    Subject string
    Body    string
}

type Worker struct {
    jobs   chan EmailJob
    sender EmailSender
}

func NewWorker(sender EmailSender, bufferSize int) *Worker {
    w := &Worker{jobs: make(chan EmailJob, bufferSize), sender: sender}
    go w.run()
    return w
}

func (w *Worker) Enqueue(job EmailJob) {
    w.jobs <- job // blocks if buffer full — backpressure
}

func (w *Worker) run() {
    for job := range w.jobs {
        if err := w.sender.Send(context.Background(), job); err != nil {
            slog.Error("email send failed", "to", job.To, "err", err)
        }
    }
}
```

Level 2 — transactional outbox (durable, exactly-once):
```go
// internal/outbox/outbox.go
// 1. Insert event into outbox table in same DB transaction as business write.
// 2. Background worker polls outbox, sends event, marks delivered.
// 3. Consumer is idempotent (deduplicates by event ID).

func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    return s.db.WithTx(ctx, func(tx *sqlx.Tx) (*Order, error) {
        order, err := s.repo.InsertOrder(ctx, tx, req)
        if err != nil {
            return nil, fmt.Errorf("insert order: %w", err)
        }
        // Outbox row committed atomically with order row.
        if err := s.outbox.Insert(ctx, tx, OutboxEvent{
            ID:      uuid.New().String(),
            Type:    "order.created",
            Payload: order,
        }); err != nil {
            return nil, fmt.Errorf("insert outbox: %w", err)
        }
        return order, nil
    })
}
```

**What to do:** Use synchronous HTTP by default. Add a goroutine channel when you have fire-and-forget work that doesn't need persistence. Add a message queue only when you need persistence, retries across process restarts, or consumer fan-out.

---

### 2.3 SQL vs NoSQL

PostgreSQL handles 95% of use cases. This section documents the remaining 5%.

```
START: What is the shape of your data?
  ├── Relational (users, orders, products with foreign keys) →
  │   POSTGRESQL. No discussion needed.
  │
  ├── Document (each record has a different schema) →
  │   ├── Can you normalize it? (Usually yes) → POSTGRESQL with JSONB.
  │   └── Truly heterogeneous, schema changes weekly →
  │       MONGODB. But confirm you actually need schemaless first.
  │
  ├── Key-value (session data, rate limits, counters, leaderboards) →
  │   REDIS. Ideal for ephemeral or time-windowed data.
  │
  ├── Time-series (IoT readings, metrics, events by timestamp) →
  │   TIMESCALEDB (PostgreSQL extension) or CLICKHOUSE.
  │   Don't build this on raw PostgreSQL — it will hurt.
  │
  └── Graph (social networks, recommendation engines) →
      NEO4J or POSTGRESQL recursive CTEs (if graph is simple).
```

**PostgreSQL + Redis is enough for 99% of Gin APIs:**

| Need | Use |
|---|---|
| Persistent data with relations | PostgreSQL |
| Session storage | Redis (TTL-based) |
| Rate limiting | Redis (sliding window counters) |
| Caching hot reads | Redis (read-through or cache-aside) |
| Full-text search | PostgreSQL `tsvector` (up to ~10M rows), then Elasticsearch |
| Geospatial queries | PostgreSQL `PostGIS` extension |
| Audit log / event log | PostgreSQL append-only table |
| Feature flags per user | PostgreSQL (simple enough for a DB query) |

**When MongoDB is actually justified:**
- The schema changes multiple times per week due to externally-driven data formats
- The data is truly document-oriented (e.g., ingesting third-party payloads of varying shapes)
- You're storing large nested objects where normalization would create 10+ tables

**When ClickHouse is justified:**
- You're running analytics over billions of rows
- Your query patterns are columnar (aggregate over all rows in a column)
- You have a dedicated analytics team that will own it

**What to do:** Start with PostgreSQL. Add Redis when you need caching or ephemeral key-value storage. Move to a specialized database only when you can measure that PostgreSQL is the bottleneck — not before.

---

### 2.4 DDD vs Simple CRUD

Most APIs are CRUD with validation. Applying DDD to them adds ceremony without value.

```
START: What does your domain model look like?
  ├── Entities with direct DB mapping, validation, maybe some computed fields →
  │   REPOSITORY PATTERN. handler → service → repository. Done.
  │
  ├── Business rules that span multiple entities + invariants to enforce →
  │   (e.g., "an order can only be fulfilled if inventory is reserved")
  │   ├── 1–3 bounded areas of complex rules →
  │   │   DOMAIN SERVICES. Add a domain layer with rich methods on entities.
  │   │   Keep it in the same package, no event sourcing.
  │   └── 4+ bounded areas with different models per area →
  │       CONSIDER BOUNDED CONTEXTS. But be honest: is this complexity
  │       in your domain, or in your current architecture?
  │
  └── Rich domain with complex state machines, aggregate roots,
      event-sourced history, multiple teams per context →
      DDD (aggregates, domain events, bounded contexts).
      Only here does the full DDD toolkit pay off.
```

**Honest test:** If you can describe your domain by listing database tables and CRUD operations on them — it's CRUD. Repository pattern is the right choice. Calling it DDD doesn't make it DDD, it just makes the code harder to read.

**Repository pattern is enough when:**
```go
// internal/user/repository.go
type Repository interface {
    GetByID(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, u *User) error
    Update(ctx context.Context, u *User) error
    Delete(ctx context.Context, id string) error
    ListByRole(ctx context.Context, role string) ([]*User, error)
}
```

**Add domain services only when:**
```go
// internal/order/domain_service.go
// Rule: "Order can only be placed if user's credit limit covers the total"
// This invariant spans Order + User + CreditAccount — not expressible in one repository.
type PlacementService struct {
    orders  OrderRepository
    users   UserRepository
    credits CreditRepository
}

func (s *PlacementService) Place(ctx context.Context, req PlaceOrderRequest) (*Order, error) {
    user, err := s.users.GetByID(ctx, req.UserID)
    if err != nil {
        return nil, fmt.Errorf("get user: %w", err)
    }
    available, err := s.credits.AvailableLimit(ctx, req.UserID)
    if err != nil {
        return nil, fmt.Errorf("get credit: %w", err)
    }
    if available < req.Total {
        return nil, domain.ErrInsufficientCredit
    }
    return s.orders.Create(ctx, &Order{UserID: req.UserID, Total: req.Total})
}
```

**What to do:** Start with handler → service → repository. Add domain services when you find invariants that cross repository boundaries and can't be expressed as a single service method. Only reach for full DDD if you have multiple bounded contexts with genuinely different models for the same concept.

---

### 2.5 Layered vs Clean vs Hexagonal Architecture

For most Gin APIs, the layered architecture is sufficient. Clean and hexagonal add indirection that only pays off in specific circumstances.

```
START: Do your tests need to swap out the web framework or database?
  ├── No → LAYERED ARCHITECTURE (handler → service → repository).
  │         Interfaces at the repository layer only. Done.
  └── Yes → Why do your tests need to swap infrastructure?
      ├── You're testing business logic without a DB (good reason) →
      │   Mock the repository interface. No clean arch needed.
      │   Layered architecture with interfaces is enough.
      └── You're planning to swap Gin for another framework (bad reason) →
          This almost never happens in practice. Don't build for it.
          If it does happen, the migration is a week of find-and-replace,
          not a multi-month refactor. Don't pay the complexity tax in advance.
```

**Layered — sufficient for 95% of Gin APIs:**

```
Handler (HTTP concerns: bind, validate, respond)
    ↓ calls
Service (business logic, orchestration)
    ↓ calls
Repository (data access, queries)
    ↓ calls
Database (PostgreSQL, Redis)
```

Interfaces live between service and repository. This gives you testability without framework lock-in ceremony.

**When hexagonal architecture pays off:**
- You genuinely have multiple adapters for the same port (e.g., HTTP handler AND gRPC handler AND CLI — all calling the same application core)
- You have strict compliance requirements that demand the domain be provably isolated from infrastructure
- Your domain logic is so complex that any coupling to persistence details would compromise correctness

**The honest trade-off:**

| | Layered | Clean / Hexagonal |
|---|---|---|
| Code to write | Less | 2–3x more (ports, adapters, mapping) |
| Onboarding | Familiar to most Go devs | Requires explanation |
| Testability | Good (mock repository interfaces) | Excellent (complete infra isolation) |
| When it pays off | Always (it's the baseline) | 3+ adapters per port, strict domain isolation |

**What to do:** Use layered. Define `Repository` interfaces in your domain package. Inject them in services. Mock them in tests. If you find yourself writing the same application logic twice (once for HTTP, once for gRPC), then ports-and-adapters starts to make sense — not before.

---

## 3. Right-Size Thinking Matrix

Use this table to calibrate the appropriate architecture for your context. Find the row that best matches your situation.

| Team Size | Stage | Traffic (RPM) | Data Complexity | Recommended Architecture | Max Pattern Score |
|---|---|---|---|---|---|
| 1–3 | MVP | < 1K | CRUD | Flat layout, direct SQL with sqlx, no repository abstraction | 5–8 |
| 1–3 | MVP | < 1K | Relational | handler → service → repository, PostgreSQL | 8–10 |
| 1–3 | Growth | 1K–10K | Relational | Same + Redis for caching hot reads, read replicas if needed | 10–13 |
| 3–8 | MVP | < 10K | Relational | Feature modules, shared repository interfaces | 12–15 |
| 3–8 | Growth | 10K–100K | Relational + events | Feature modules + async worker, Redis queue | 15–20 |
| 3–8 | Growth | 10K–100K | Complex rules | Domain services per bounded area | 16–20 |
| 8–15 | Growth | 10K–100K | Relational | Modular monolith, consider 1–2 service extractions | 18–22 |
| 8–15 | Mature | 100K+ | Mixed | Limited microservices (proven pain only), CQRS for read-heavy | 22–28 |
| 15+ | Mature | 100K+ | Event-driven | Multiple services, event sourcing for audit-critical domains | 28+ |

**How to use this table:**
1. Find the row that best matches your team size, stage, and traffic.
2. Use the "Recommended Architecture" as your starting point.
3. Use the "Max Pattern Score" as a ceiling when evaluating proposals.
4. If a proposal exceeds the ceiling, require explicit justification with measured pain evidence.

---

## 4. Pattern Selection Matrix

For each pattern: complexity cost, the actual signal that justifies it, the simpler alternative, and the gate question that must be answered before adoption.

### Repository Pattern

| | |
|---|---|
| Complexity cost | 1 |
| What it solves | Decouples business logic from data access; enables mocking in tests |
| Simpler alternative | Direct sqlx calls in service (acceptable for 1–3 devs, MVP only) |
| You need it when | You want to mock the DB in unit tests, or you have more than one data source |
| You DON'T need it if | You're building a prototype or internal tool where tests aren't required |

```go
// internal/user/repository.go
type Repository interface {
    GetByID(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, u *User) error
}

type postgresRepository struct{ db *sqlx.DB }

func (r *postgresRepository) GetByID(ctx context.Context, id string) (*User, error) {
    var u User
    err := r.db.GetContext(ctx, &u, `SELECT * FROM users WHERE id = $1`, id)
    return &u, err
}
```

---

### Service Layer

| | |
|---|---|
| Complexity cost | 1 |
| What it solves | Business logic lives outside handlers; testable without HTTP |
| Simpler alternative | Logic directly in handler (acceptable for < 5 endpoints, prototypes) |
| You need it when | You have business logic that isn't just "validate + persist" |
| You DON'T need it if | Your handler literally only validates input, calls one repo method, returns result |

---

### CQRS (Command Query Responsibility Segregation)

| | |
|---|---|
| Complexity cost | 4 |
| What it solves | Separate read and write models; optimize each independently |
| Simpler alternative | Single model with a `ListXxx` method that returns a DTO (covers 80% of cases) |
| You need it when | Read queries require joining 5+ tables and are 10x the volume of writes |
| You DON'T need it if | Your reads and writes operate on the same data shape |

```go
// CQRS only makes sense when the read model is genuinely different.
// Write side: normalized domain model with validation and invariants.
// Read side: denormalized view model optimized for the query.

// Command side — domain model
type CreateOrderCommand struct {
    UserID string
    Items  []OrderItem
}

// Query side — view model (might be populated from a materialized view)
type OrderSummaryView struct {
    ID          string
    UserName    string   // joined from users table
    TotalItems  int
    TotalAmount float64
    Status      string
    CreatedAt   time.Time
}
```

---

### Event Sourcing

| | |
|---|---|
| Complexity cost | 5 |
| What it solves | Complete audit trail; rebuild any past state; temporal queries |
| Simpler alternative | Append-only audit log table in PostgreSQL (handles 90% of audit requirements) |
| You need it when | You need to reconstruct state at any point in time AND the domain has complex state transitions AND the team has operational experience running event stores |
| You DON'T need it if | You need "an audit log" — that's a PostgreSQL table, not event sourcing |

```go
// Before reaching for event sourcing, try this — it handles most audit needs:
// CREATE TABLE audit_log (
//   id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
//   table_name TEXT NOT NULL,
//   record_id  TEXT NOT NULL,
//   action     TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
//   old_values JSONB,
//   new_values JSONB,
//   actor_id   TEXT,
//   created_at TIMESTAMPTZ DEFAULT now()
// );
```

---

### Saga / Choreography

| | |
|---|---|
| Complexity cost | 4 |
| What it solves | Distributed transactions across multiple services without 2PC |
| Simpler alternative | Database transactions (if in same DB) or a single service that owns the workflow |
| You need it when | You have a multi-step workflow that spans 3+ services and partial failure must be compensated |
| You DON'T need it if | The workflow can live in a single service — even if it calls multiple repos |

---

### Circuit Breaker

| | |
|---|---|
| Complexity cost | 2 |
| What it solves | Prevents cascade failure when a downstream dependency is unhealthy |
| Simpler alternative | Timeout + retry with backoff (covers most cases) |
| You need it when | You have a dependency that fails in a way that ties up goroutines (slow failures, not fast failures) |
| You DON'T need it if | Your downstream returns errors quickly (fast failures don't cascade) |

```go
// github.com/sony/gobreaker is the standard Go implementation
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "payment-service",
    MaxRequests: 5,               // allow 5 requests in half-open state
    Interval:    10 * time.Second,
    Timeout:     30 * time.Second,
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        failRatio := float64(counts.TotalFailures) / float64(counts.Requests)
        return counts.Requests >= 10 && failRatio >= 0.6
    },
})

result, err := cb.Execute(func() (interface{}, error) {
    return paymentClient.Charge(ctx, req)
})
```

---

### Feature Flags

| | |
|---|---|
| Complexity cost | 2 |
| What it solves | Deploy code without activating it; gradual rollouts; A/B testing |
| Simpler alternative | Environment variables (for simple on/off by deployment) |
| You need it when | You need per-user or per-percentage rollouts, or to roll back a feature without redeploying |
| You DON'T need it if | You only need to toggle a feature at deploy time — use an env var |

```go
// Simple PostgreSQL-backed feature flag — no external service needed at small scale
type FeatureFlag struct {
    Name       string
    Enabled    bool
    RolloutPct int    // 0-100
    UserIDs    []string // allowlist
}

func (s *FeatureService) IsEnabled(ctx context.Context, flag string, userID string) bool {
    f, err := s.repo.GetFlag(ctx, flag)
    if err != nil || !f.Enabled {
        return false
    }
    if slices.Contains(f.UserIDs, userID) {
        return true
    }
    // Deterministic rollout: hash(userID + flagName) % 100 < rolloutPct
    h := fnv.New32a()
    h.Write([]byte(userID + flag))
    return int(h.Sum32()%100) < f.RolloutPct
}
```

---

### API Gateway

| | |
|---|---|
| Complexity cost | 3 |
| What it solves | Single entry point for routing, auth, rate limiting, SSL termination across multiple services |
| Simpler alternative | Nginx or Caddy reverse proxy (handles SSL termination and routing without custom logic) |
| You need it when | You have 3+ services with different auth models, and you need request transformation or fan-out |
| You DON'T need it if | You have a single service or a monolith — a reverse proxy is enough |

---

### Service Mesh

| | |
|---|---|
| Complexity cost | 5 |
| What it solves | mTLS between services, observability at the network layer, traffic management, retries |
| Simpler alternative | Application-level TLS + OpenTelemetry + circuit breaker (covers most needs) |
| You need it when | You have 10+ services, strict mTLS compliance requirements, and a dedicated platform team |
| You DON'T need it if | You have fewer than 10 services, or no platform engineer to own it |

---

## 5. You Don't Need This Yet — Gates

For each high-complexity pattern, ALL listed conditions must be true before adoption is justified. If any are false, use the simpler alternative.

### Microservices

All 5 must be true:

- [ ] You have 3+ independent teams that need to deploy on different schedules and are currently blocked
- [ ] You have measured (not estimated) that a specific module has an independent scaling bottleneck
- [ ] You have working distributed tracing across all services (not just logging)
- [ ] You have a CI/CD pipeline that can build, test, and deploy each service independently in under 10 minutes
- [ ] You have on-call runbooks for each service and a tested rollback procedure

If any is false: build a modular monolith. Clean internal package boundaries give you 80% of the benefit at 10% of the cost.

### CQRS

All 4 must be true:

- [ ] Read volume is measurably 10x+ write volume (check your query logs)
- [ ] The read model requires data that is inconvenient to compute at query time (multiple joins, aggregations)
- [ ] You have profiled PostgreSQL and confirmed it is the bottleneck — not your Go code or network
- [ ] Your team has built CQRS before and understands eventual consistency implications

If any is false: add a read-optimized query method to your repository. A well-indexed PostgreSQL view handles most "CQRS-shaped problems" without the overhead.

### Event Sourcing

All 4 must be true:

- [ ] You have a legal or compliance requirement to reproduce system state at any past point in time
- [ ] Your domain has complex state machines where the transition history matters, not just current state
- [ ] Your team has operational experience running an event store (Kafka, EventStoreDB) in production
- [ ] You have designed your aggregates and projections and the team agrees on the boundaries

If any is false: use a PostgreSQL append-only audit log + a standard CRUD domain model. It solves 90% of audit requirements with 10% of the complexity.

### Service Mesh (Istio, Linkerd)

All 3 must be true:

- [ ] You have 10+ services in production with active traffic
- [ ] You have a dedicated platform engineer who owns the mesh configuration
- [ ] You have a compliance requirement for mTLS between services that cannot be satisfied by application-level TLS

If any is false: handle TLS at the load balancer, use OpenTelemetry for observability, and implement circuit breakers at the application layer. You get most of the value with zero mesh overhead.

---

**Final rule:** Architecture is not a competition. The system that ships, that the team can maintain, and that can be extended by a new hire in their first week — that is the right architecture. Complexity is a debt. Spend it wisely.
