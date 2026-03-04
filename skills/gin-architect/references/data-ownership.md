# Data Ownership Reference

How services own and share data in Go architectures. Covers database-per-service, shared data access patterns, data synchronization strategies, and the API composition pattern. Load when migrating from monolith to services, evaluating service extraction, or deciding on data boundaries.

> **Default stance:** Start with a single database. Split ONLY when you have a genuine reason (different scaling needs, team autonomy, technology mismatch). Premature data splitting creates distributed transaction nightmares.

## Table of Contents

1. [When to Split Data](#1-when-to-split-data)
2. [Database-Per-Service Pattern](#2-database-per-service-pattern)
3. [API Composition](#3-api-composition)
4. [Data Sync Strategies](#4-data-sync-strategies)
5. [Shared Reference Data](#5-shared-reference-data)
6. [Migration Path: Monolith to Split](#6-migration-path-monolith-to-split)
7. [Cross-Skill References](#7-cross-skill-references)

---

## 1. When to Split Data

### Gate — VERY HIGH cost (5/5)

Database-per-service introduces distributed transactions, data consistency challenges, eventual consistency tradeoffs, and operational overhead. Before proceeding, all conditions in the chosen branch must be true.

### Decision Tree

```
START: Should this module have its own database?
  │
  ├── Same team deploys everything?
  │     └── Shared DB is fine. STOP. Do not split.
  │
  └── Different teams, different deploy cycles?
        │
        ├── Data is tightly coupled (JOINs between modules)?
        │     └── Shared DB. Splitting will hurt more than help.
        │
        └── Data has clear boundaries (no cross-module JOINs)?
              │
              ├── Different scaling needs?
              │     └── Split → Database-per-service.
              │
              └── Same scaling needs?
                    └── Schema-per-service (same PostgreSQL, different schemas).
```

### Signals that splitting is premature

- You need `JOIN` between the modules in any query
- One team owns both modules
- Both modules deploy together on the same schedule
- You haven't profiled a performance bottleneck that requires isolation

### Signals that splitting may be justified

- Teams are blocked waiting on each other's migrations
- One module needs PostgreSQL and another needs a document store
- One module receives 100x the write load of others and needs independent scaling
- Regulatory requirement for data isolation (e.g., PII must reside in a separate database)

---

## 2. Database-Per-Service Pattern

### Architecture

```
┌──────────────────┐     ┌──────────────────┐
│   User Service   │     │  Order Service   │
│    (Gin API)     │     │    (Gin API)     │
└────────┬─────────┘     └────────┬─────────┘
         │                        │
┌────────▼─────────┐     ┌────────▼─────────┐
│    user_db       │     │    order_db      │
│   (PostgreSQL)   │     │   (PostgreSQL)   │
└──────────────────┘     └──────────────────┘
```

### Rules

- Service A **never** queries Service B's database directly — no shared connection strings
- All cross-service data access goes through APIs (HTTP or gRPC)
- Each service owns its schema and runs its own migrations
- No foreign keys across service boundaries — enforce integrity via application logic

### Repository scoped to one service

```go
// internal/user/repository/postgres/user_repository.go
type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    const q = `SELECT id, email, name FROM users WHERE id = $1`
    row := r.db.QueryRowContext(ctx, q, id)

    var u domain.User
    if err := row.Scan(&u.ID, &u.Email, &u.Name); err != nil {
        return nil, fmt.Errorf("UserRepository.GetByID: %w", err)
    }
    return &u, nil
}
```

---

## 3. API Composition

When a client needs data from multiple services, an API gateway or BFF (Backend for Frontend) fans out requests and merges results. Never let the client call multiple services directly — that leaks internal topology.

### Fan-out with errgroup

```go
// internal/gateway/handler/order_details_handler.go
package handler

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "golang.org/x/sync/errgroup"

    "myapp/internal/gateway/client"
)

type OrderDetailsHandler struct {
    orders client.OrderClient
    users  client.UserClient
}

func NewOrderDetailsHandler(orders client.OrderClient, users client.UserClient) *OrderDetailsHandler {
    return &OrderDetailsHandler{orders: orders, users: users}
}

func (h *OrderDetailsHandler) Get(c *gin.Context) {
    orderID := c.Param("id")

    // Fetch order first — we need UserID before we can fetch the user.
    order, err := h.orders.GetByID(c.Request.Context(), orderID)
    if err != nil {
        slog.ErrorContext(c.Request.Context(), "fetch order", "err", err)
        c.JSON(http.StatusBadGateway, gin.H{"error": "order service unavailable"})
        return
    }

    // Fan-out: enrich with data from other services in parallel.
    g, ctx := errgroup.WithContext(c.Request.Context())

    var user *client.User
    g.Go(func() error {
        var err error
        user, err = h.users.GetByID(ctx, order.UserID)
        return err
    })

    if err := g.Wait(); err != nil {
        slog.ErrorContext(c.Request.Context(), "enrich order", "err", err)
        c.JSON(http.StatusBadGateway, gin.H{"error": "enrichment failed"})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "order": order,
        "user":  user,
    })
}
```

### Client interface (typed, testable)

```go
// internal/gateway/client/order_client.go
package client

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
)

type Order struct {
    ID     string `json:"id"`
    UserID string `json:"user_id"`
    Total  int64  `json:"total"`
}

type OrderClient interface {
    GetByID(ctx context.Context, id string) (*Order, error)
}

type httpOrderClient struct {
    base string
    http *http.Client
}

func NewHTTPOrderClient(baseURL string, c *http.Client) OrderClient {
    return &httpOrderClient{base: baseURL, http: c}
}

func (c *httpOrderClient) GetByID(ctx context.Context, id string) (*Order, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, c.base+"/orders/"+id, nil)
    if err != nil {
        return nil, fmt.Errorf("OrderClient.GetByID build request: %w", err)
    }

    resp, err := c.http.Do(req)
    if err != nil {
        return nil, fmt.Errorf("OrderClient.GetByID: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("OrderClient.GetByID: upstream returned %d", resp.StatusCode)
    }

    var o Order
    if err := json.NewDecoder(resp.Body).Decode(&o); err != nil {
        return nil, fmt.Errorf("OrderClient.GetByID decode: %w", err)
    }
    return &o, nil
}
```

---

## 4. Data Sync Strategies

| Strategy | Use when | Consistency | Complexity |
|---|---|---|---|
| API call on demand | Low traffic, freshness required | Strong | LOW |
| Event-driven (message broker) | Eventually consistent is acceptable | Eventual | MEDIUM |
| CDC (Change Data Capture) | Real-time sync without app changes | Eventual | HIGH |
| Materialized view (CQRS) | Read-heavy, complex joins across services | Eventual | HIGH |

**Default:** API call on demand. Move to event-driven only when measured latency or coupling becomes a real problem.

### Event-driven example (order.created)

Order service publishes an event. User service consumes it to maintain a local read model.

```go
// order service — publish after successful insert
func (s *OrderService) Create(ctx context.Context, cmd CreateOrderCmd) (*Order, error) {
    order, err := s.repo.Insert(ctx, cmd)
    if err != nil {
        return nil, fmt.Errorf("OrderService.Create: %w", err)
    }

    evt := OrderCreatedEvent{
        OrderID: order.ID,
        UserID:  order.UserID,
        Total:   order.Total,
    }
    if err := s.publisher.Publish(ctx, "order.created", evt); err != nil {
        // Log and continue — use outbox pattern for guaranteed delivery.
        slog.ErrorContext(ctx, "publish order.created", "err", err, "order_id", order.ID)
    }

    return order, nil
}
```

```go
// user service — consume order.created to update local count
func (c *OrderCountConsumer) Handle(ctx context.Context, msg OrderCreatedEvent) error {
    if err := c.repo.IncrementOrderCount(ctx, msg.UserID); err != nil {
        return fmt.Errorf("OrderCountConsumer.Handle: %w", err)
    }
    slog.InfoContext(ctx, "order count updated", "user_id", msg.UserID)
    return nil
}
```

For guaranteed delivery, combine with the Transactional Outbox pattern (see `data-patterns.md`).

---

## 5. Shared Reference Data

Reference data: countries, currencies, product categories, feature flags. Low write frequency, high read frequency.

### Three options

| Option | When to use | Trade-off |
|---|---|---|
| **Each service keeps a copy** (synced via events) | High read volume, tolerates eventual consistency | Stale data window; sync complexity |
| **Dedicated reference data service** | Many services need it, central ownership matters | Extra hop per request; single point of failure |
| **Config / static data embedded in code** | Data rarely changes (< once per release) | Redeploy to update; zero latency |

### Option C — embedded static data (prefer this first)

```go
// internal/reference/currency.go
package reference

// Currencies is a static list compiled into the binary.
// Update requires a redeploy. Acceptable when currencies change < once per year.
var Currencies = map[string]string{
    "BRL": "Brazilian Real",
    "USD": "US Dollar",
    "EUR": "Euro",
}

func ValidCurrency(code string) bool {
    _, ok := Currencies[code]
    return ok
}
```

### Option A — local copy synced via events

```go
// internal/reference/repository/currency_cache.go
package repository

import (
    "context"
    "sync"
)

type CurrencyCache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *CurrencyCache) Set(code, name string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[code] = name
}

func (c *CurrencyCache) Get(ctx context.Context, code string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    name, ok := c.data[code]
    return name, ok
}
```

---

## 6. Migration Path: Monolith to Split

Migrate incrementally. Never attempt a big-bang split.

### Steps

**Step 1 — Start: single shared PostgreSQL**
```
┌──────────────────────────────┐
│         Monolith API         │
│          (Gin)               │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│         shared_db            │
│  tables: users, orders, ...  │
└──────────────────────────────┘
```

**Step 2 — Separate schemas (same PostgreSQL instance)**
```sql
-- Run once in a migration
CREATE SCHEMA IF NOT EXISTS users;
CREATE SCHEMA IF NOT EXISTS orders;

-- Move tables into schemas
ALTER TABLE users SET SCHEMA users;
ALTER TABLE orders SET SCHEMA orders;
```

Use schema prefix in repositories:

```go
// internal/user/repository/postgres/user_repository.go
const schema = "users"

func (r *UserRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    query := fmt.Sprintf(`SELECT id, email, name FROM %s.users WHERE id = $1`, schema)
    row := r.db.QueryRowContext(ctx, query, id)

    var u domain.User
    if err := row.Scan(&u.ID, &u.Email, &u.Name); err != nil {
        return nil, fmt.Errorf("UserRepository.GetByID: %w", err)
    }
    return &u, nil
}
```

**Step 3 — Replace cross-schema JOINs with service calls**

Before:
```sql
-- Bad: cross-schema JOIN couples both modules
SELECT o.id, u.email
FROM orders.orders o
JOIN users.users u ON u.id = o.user_id
WHERE o.id = $1;
```

After — fetch separately, join in application:
```go
order, _ := orderRepo.GetByID(ctx, orderID)
user, _  := userClient.GetByID(ctx, order.UserID)
```

**Step 4 — Move schemas to separate databases**

Change the connection string per service. No application code changes if repositories use the schema constant.

```go
// cmd/user-service/main.go
db, err := sql.Open("pgx", os.Getenv("USER_DB_DSN"))

// cmd/order-service/main.go
db, err := sql.Open("pgx", os.Getenv("ORDER_DB_DSN"))
```

**Step 5 — Deploy as separate services**

Each service runs as its own binary, owns its own database, communicates via HTTP/gRPC. Apply API composition (Section 3) where clients need combined views.

---

## 7. Cross-Skill References

| Reference | Why |
|---|---|
| `system-design.md` | Bounded context analysis and service boundary design |
| `data-patterns.md` | CQRS, Transactional Outbox, Saga pattern for distributed writes |
| `messaging-patterns.md` | Event-driven sync implementation with RabbitMQ / NATS |
| `gin-psql-dba` skill | Schema design, migrations, and PostgreSQL schema isolation |
