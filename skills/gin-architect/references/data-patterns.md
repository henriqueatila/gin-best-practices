# Data Patterns Reference

High-complexity patterns for Go Gin APIs that need advanced data handling. Each entry has a **Gate** — a checklist of conditions that must ALL be true before adopting. If even one condition is false, use the simpler alternative.

**Default stance:** PostgreSQL + clean repositories handles more than you think. Add patterns only when you have measured evidence that you need them.

## Table of Contents

1. [CQRS](#1-cqrs-command-query-responsibility-segregation)
2. [Event Sourcing](#2-event-sourcing)
3. [Saga Pattern](#3-saga-pattern-orchestration-vs-choreography)
4. [Transactional Outbox](#4-transactional-outbox)
5. [Read Replicas](#5-read-replicas)
6. [Pattern Comparison Table](#6-pattern-comparison-table)

---

## 1. CQRS (Command Query Responsibility Segregation)

### Gate — all must be true

- [ ] Read and write models are fundamentally different shapes (not just different subsets of the same row)
- [ ] Read volume is 10x+ write volume AND the read path is the bottleneck
- [ ] A single PostgreSQL primary + read replica is not enough
- [ ] Your team has at least 3 engineers who understand the pattern

**Simpler alternative first:** Separate `ReadUser()` / `WriteUser()` methods in the same repository. Add a PostgreSQL materialized view for denormalized reads. This covers the vast majority of "reads and writes look different" cases without splitting infrastructure.

```sql
-- Read-optimized view — refresh on schedule or via trigger
CREATE MATERIALIZED VIEW order_summary AS
SELECT
    o.id,
    o.user_id,
    u.name  AS user_name,
    COUNT(oi.id) AS item_count,
    SUM(oi.price * oi.quantity) AS total_amount,
    o.status,
    o.created_at
FROM orders o
JOIN users u  ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id, o.user_id, u.name, o.status, o.created_at;

CREATE UNIQUE INDEX ON order_summary(id);
```

### Go Implementation

```go
// internal/order/command.go
package order

import (
    "context"
    "fmt"
    "time"
)

// --- Write side: normalized domain model ---

type CreateOrderCommand struct {
    UserID string
    Items  []OrderItemInput
}

type OrderItemInput struct {
    ProductID string
    Quantity  int
    Price     int64 // cents
}

type OrderWriteRepo interface {
    Create(ctx context.Context, userID string, items []OrderItemInput) (string, error)
}

type OrderCommandHandler struct {
    repo OrderWriteRepo
}

func NewOrderCommandHandler(repo OrderWriteRepo) *OrderCommandHandler {
    return &OrderCommandHandler{repo: repo}
}

func (h *OrderCommandHandler) CreateOrder(ctx context.Context, cmd CreateOrderCommand) (string, error) {
    if len(cmd.Items) == 0 {
        return "", fmt.Errorf("order must have at least one item")
    }
    id, err := h.repo.Create(ctx, cmd.UserID, cmd.Items)
    if err != nil {
        return "", fmt.Errorf("create order: %w", err)
    }
    return id, nil
}
```

```go
// internal/order/query.go
package order

import (
    "context"
    "fmt"
    "time"

    "github.com/jmoiron/sqlx"
)

// --- Read side: denormalized view ---

type OrderView struct {
    ID          string    `db:"id"`
    UserName    string    `db:"user_name"`
    ItemCount   int       `db:"item_count"`
    TotalAmount int64     `db:"total_amount"`
    Status      string    `db:"status"`
    CreatedAt   time.Time `db:"created_at"`
}

type OrderQueryHandler struct {
    db *sqlx.DB // can point to a read replica
}

func NewOrderQueryHandler(db *sqlx.DB) *OrderQueryHandler {
    return &OrderQueryHandler{db: db}
}

func (h *OrderQueryHandler) ListOrders(ctx context.Context, userID string, page, limit int) ([]OrderView, error) {
    var views []OrderView
    q := `SELECT * FROM order_summary WHERE user_id = $1 ORDER BY created_at DESC LIMIT $2 OFFSET $3`
    if err := h.db.SelectContext(ctx, &views, q, userID, limit, (page-1)*limit); err != nil {
        return nil, fmt.Errorf("list orders: %w", err)
    }
    return views, nil
}
```

**Wire in handler:**

```go
// internal/order/handler.go
package order

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
)

type Handler struct {
    cmd   *OrderCommandHandler
    query *OrderQueryHandler
}

func (h *Handler) Create(c *gin.Context) {
    var cmd CreateOrderCommand
    if err := c.ShouldBindJSON(&cmd); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    id, err := h.cmd.CreateOrder(c.Request.Context(), cmd)
    if err != nil {
        slog.ErrorContext(c.Request.Context(), "create order failed", "err", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }
    c.JSON(http.StatusCreated, gin.H{"id": id})
}
```

---

## 2. Event Sourcing

### Gate — all must be true

- [ ] Full audit trail of every state change is a **business or compliance requirement** (not "nice to have")
- [ ] You need temporal queries: "what was the state at time T?"
- [ ] Multiple different projections of the same data are needed now (not speculatively)
- [ ] You accept: event store ops, projection rebuilds, eventual consistency in all reads, and snapshot management
- [ ] Team has experience operating event-sourced systems

**Simpler alternative first:** An `audit_log` table with `(entity_id, entity_type, action, actor_id, diff jsonb, created_at)` covers 95% of audit requirements. Query it with plain SQL. Zero new infrastructure.

```sql
CREATE TABLE audit_log (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id   UUID        NOT NULL,
    entity_type TEXT        NOT NULL,
    action      TEXT        NOT NULL,  -- 'created', 'updated', 'deleted'
    actor_id    UUID,
    diff        JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON audit_log(entity_id, created_at DESC);
```

### Go Implementation

```go
// internal/eventsource/store.go
package eventsource

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/jmoiron/sqlx"
)

// Event is a single immutable fact stored in the event store.
type Event struct {
    ID            int64           `db:"id"`
    AggregateID   string          `db:"aggregate_id"`
    AggregateType string          `db:"aggregate_type"`
    EventType     string          `db:"event_type"`
    Payload       json.RawMessage `db:"payload"`
    Version       int             `db:"version"`
    OccurredAt    time.Time       `db:"occurred_at"`
}

// EventStore persists and retrieves events for an aggregate.
type EventStore struct {
    db *sqlx.DB
}

func NewEventStore(db *sqlx.DB) *EventStore { return &EventStore{db: db} }

// Append inserts events for an aggregate. Uses optimistic concurrency via expectedVersion.
func (s *EventStore) Append(ctx context.Context, aggregateID string, events []Event, expectedVersion int) error {
    tx, err := s.db.BeginTxx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback()

    var currentVersion int
    err = tx.QueryRowContext(ctx,
        `SELECT COALESCE(MAX(version), 0) FROM events WHERE aggregate_id = $1`,
        aggregateID,
    ).Scan(&currentVersion)
    if err != nil {
        return fmt.Errorf("read version: %w", err)
    }
    if currentVersion != expectedVersion {
        return fmt.Errorf("optimistic lock: expected version %d, got %d", expectedVersion, currentVersion)
    }

    for i, e := range events {
        _, err = tx.ExecContext(ctx,
            `INSERT INTO events (aggregate_id, aggregate_type, event_type, payload, version, occurred_at)
             VALUES ($1, $2, $3, $4, $5, NOW())`,
            aggregateID, e.AggregateType, e.EventType, e.Payload, expectedVersion+i+1,
        )
        if err != nil {
            return fmt.Errorf("insert event: %w", err)
        }
    }
    return tx.Commit()
}

// Load retrieves all events for an aggregate, ordered by version.
func (s *EventStore) Load(ctx context.Context, aggregateID string) ([]Event, error) {
    var events []Event
    err := s.db.SelectContext(ctx, &events,
        `SELECT * FROM events WHERE aggregate_id = $1 ORDER BY version ASC`,
        aggregateID,
    )
    if err != nil {
        return nil, fmt.Errorf("load events: %w", err)
    }
    return events, nil
}
```

```go
// internal/eventsource/aggregate.go
package eventsource

import "encoding/json"

// Aggregate reconstructs state by replaying events.
type OrderAggregate struct {
    ID      string
    Status  string
    Version int
}

func (a *OrderAggregate) Apply(e Event) {
    switch e.EventType {
    case "OrderCreated":
        var p struct{ Status string }
        _ = json.Unmarshal(e.Payload, &p)
        a.ID = e.AggregateID
        a.Status = p.Status
    case "OrderShipped":
        a.Status = "shipped"
    case "OrderCancelled":
        a.Status = "cancelled"
    }
    a.Version = e.Version
}

func RebuildOrder(events []Event) OrderAggregate {
    var agg OrderAggregate
    for _, e := range events {
        agg.Apply(e)
    }
    return agg
}
```

**Snapshot optimization** (add when replay of >500 events becomes slow):

```go
// internal/eventsource/snapshot.go
package eventsource

import (
    "context"
    "encoding/json"
    "fmt"

    "github.com/jmoiron/sqlx"
)

type Snapshot struct {
    AggregateID string          `db:"aggregate_id"`
    Version     int             `db:"version"`
    State       json.RawMessage `db:"state"`
}

func SaveSnapshot(ctx context.Context, db *sqlx.DB, s Snapshot) error {
    _, err := db.ExecContext(ctx,
        `INSERT INTO snapshots (aggregate_id, version, state)
         VALUES ($1, $2, $3)
         ON CONFLICT (aggregate_id) DO UPDATE SET version = $2, state = $3`,
        s.AggregateID, s.Version, s.State,
    )
    if err != nil {
        return fmt.Errorf("save snapshot: %w", err)
    }
    return nil
}
```

**Schema:**

```sql
CREATE TABLE events (
    id             BIGSERIAL PRIMARY KEY,
    aggregate_id   UUID        NOT NULL,
    aggregate_type TEXT        NOT NULL,
    event_type     TEXT        NOT NULL,
    payload        JSONB       NOT NULL,
    version        INT         NOT NULL,
    occurred_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (aggregate_id, version)
);

CREATE TABLE snapshots (
    aggregate_id UUID PRIMARY KEY,
    version      INT  NOT NULL,
    state        JSONB NOT NULL
);
```

---

## 3. Saga Pattern (Orchestration vs Choreography)

### Gate — all must be true

- [ ] A business transaction spans multiple services or databases
- [ ] Each step can fail independently and may have already partially committed
- [ ] You need compensating actions (rollbacks that can't use a single DB transaction)
- [ ] A single `BEGIN / COMMIT` across one database is NOT sufficient

**Simpler alternative first:** If all data is in one PostgreSQL database, use a database transaction — it's ACID, it rolls back automatically, and it requires zero new code. If two services are involved, call them sequentially with an explicit compensation call on error:

```go
// Two-service sequential with manual compensation — no saga needed
func (s *CheckoutService) Checkout(ctx context.Context, orderID string) error {
    if err := s.inventoryClient.Reserve(ctx, orderID); err != nil {
        return fmt.Errorf("reserve inventory: %w", err)
    }
    if err := s.paymentClient.Charge(ctx, orderID); err != nil {
        _ = s.inventoryClient.Release(ctx, orderID) // compensate
        return fmt.Errorf("charge payment: %w", err)
    }
    return nil
}
```

### Orchestrator Saga (Go)

Use an orchestrator when you have 3+ steps or need durable state across retries.

```go
// internal/saga/checkout_saga.go
package saga

import (
    "context"
    "fmt"
    "log/slog"
    "time"
)

type SagaStep struct {
    Name      string
    Execute   func(ctx context.Context, sagaID string) error
    Compensate func(ctx context.Context, sagaID string) error
}

type SagaOrchestrator struct {
    steps []SagaStep
    store SagaStateStore
}

type SagaStateStore interface {
    SaveProgress(ctx context.Context, sagaID string, completedStep int) error
    LoadProgress(ctx context.Context, sagaID string) (int, error)
    MarkFailed(ctx context.Context, sagaID string, reason string) error
    MarkCompleted(ctx context.Context, sagaID string) error
}

func (o *SagaOrchestrator) Run(ctx context.Context, sagaID string) error {
    lastCompleted, _ := o.store.LoadProgress(ctx, sagaID)
    completed := make([]int, 0, len(o.steps))

    for i, step := range o.steps {
        if i < lastCompleted {
            completed = append(completed, i) // already done in prior run
            continue
        }
        slog.InfoContext(ctx, "saga step executing", "saga_id", sagaID, "step", step.Name)
        if err := step.Execute(ctx, sagaID); err != nil {
            slog.ErrorContext(ctx, "saga step failed", "saga_id", sagaID, "step", step.Name, "err", err)
            _ = o.store.MarkFailed(ctx, sagaID, err.Error())
            // Compensate in reverse order.
            for j := len(completed) - 1; j >= 0; j-- {
                si := completed[j]
                if o.steps[si].Compensate != nil {
                    if cerr := o.steps[si].Compensate(ctx, sagaID); cerr != nil {
                        slog.ErrorContext(ctx, "compensation failed", "step", o.steps[si].Name, "err", cerr)
                    }
                }
            }
            return fmt.Errorf("saga %s failed at step %s: %w", sagaID, step.Name, err)
        }
        completed = append(completed, i)
        _ = o.store.SaveProgress(ctx, sagaID, i+1)
    }

    return o.store.MarkCompleted(ctx, sagaID)
}
```

**PostgreSQL state store:**

```sql
CREATE TABLE saga_state (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saga_type       TEXT        NOT NULL,
    status          TEXT        NOT NULL DEFAULT 'running', -- running|completed|failed
    completed_steps INT         NOT NULL DEFAULT 0,
    failure_reason  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 4. Transactional Outbox

### Gate — all must be true

- [ ] You need to atomically update a database record AND publish a message/event
- [ ] "At-least-once" delivery is a requirement (losing messages is not acceptable)
- [ ] You cannot use a distributed transaction (2PC) or it is too expensive

**Simpler alternative:** If you can tolerate occasional duplicate messages but not lost messages: write to DB, then publish. If publish fails, a background job retries. This is the outbox pattern without a formal table — acceptable for low-criticality events.

### Schema

```sql
CREATE TABLE outbox (
    id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    topic        TEXT        NOT NULL,
    payload      JSONB       NOT NULL,
    status       TEXT        NOT NULL DEFAULT 'pending', -- pending|sent|failed
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at TIMESTAMPTZ
);
CREATE INDEX ON outbox(status, created_at) WHERE status = 'pending';
```

### Writer — atomic DB update + outbox insert

```go
// internal/order/repository.go
package order

import (
    "context"
    "encoding/json"
    "fmt"

    "github.com/jmoiron/sqlx"
)

func (r *Repository) CreateWithOutbox(ctx context.Context, cmd CreateOrderCommand) (string, error) {
    tx, err := r.db.BeginTxx(ctx, nil)
    if err != nil {
        return "", fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback()

    var id string
    err = tx.QueryRowContext(ctx,
        `INSERT INTO orders (user_id, status) VALUES ($1, 'pending') RETURNING id`,
        cmd.UserID,
    ).Scan(&id)
    if err != nil {
        return "", fmt.Errorf("insert order: %w", err)
    }

    payload, _ := json.Marshal(map[string]string{"order_id": id, "user_id": cmd.UserID})
    _, err = tx.ExecContext(ctx,
        `INSERT INTO outbox (topic, payload) VALUES ('order.created', $1)`,
        payload,
    )
    if err != nil {
        return "", fmt.Errorf("insert outbox: %w", err)
    }

    return id, tx.Commit()
}
```

### Publisher — poll and dispatch

```go
// internal/outbox/publisher.go
package outbox

import (
    "context"
    "encoding/json"
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
)

// MessageBroker is any system that accepts topic + payload (Kafka, SQS, Redis Streams, etc.)
type MessageBroker interface {
    Publish(ctx context.Context, topic string, payload json.RawMessage) error
}

type OutboxPublisher struct {
    db     *sqlx.DB
    broker MessageBroker
}

func NewOutboxPublisher(db *sqlx.DB, broker MessageBroker) *OutboxPublisher {
    return &OutboxPublisher{db: db, broker: broker}
}

// Run polls the outbox every interval, publishing pending messages.
func (p *OutboxPublisher) Run(ctx context.Context, interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    for {
        select {
        case <-ticker.C:
            if err := p.processBatch(ctx); err != nil {
                slog.ErrorContext(ctx, "outbox publish failed", "err", err)
            }
        case <-ctx.Done():
            return
        }
    }
}

type outboxRow struct {
    ID      string          `db:"id"`
    Topic   string          `db:"topic"`
    Payload json.RawMessage `db:"payload"`
}

func (p *OutboxPublisher) processBatch(ctx context.Context) error {
    var rows []outboxRow
    err := p.db.SelectContext(ctx, &rows,
        `SELECT id, topic, payload FROM outbox WHERE status = 'pending'
         ORDER BY created_at LIMIT 100 FOR UPDATE SKIP LOCKED`,
    )
    if err != nil {
        return fmt.Errorf("select outbox: %w", err)
    }

    for _, row := range rows {
        pubErr := p.broker.Publish(ctx, row.Topic, row.Payload)
        if pubErr != nil {
            slog.ErrorContext(ctx, "broker publish failed", "id", row.ID, "err", pubErr)
            _, _ = p.db.ExecContext(ctx,
                `UPDATE outbox SET status = 'failed' WHERE id = $1`, row.ID)
            continue
        }
        _, _ = p.db.ExecContext(ctx,
            `UPDATE outbox SET status = 'sent', processed_at = NOW() WHERE id = $1`, row.ID)
    }
    return nil
}
```

**Idempotent consumers:** Each consumer must deduplicate by `outbox.id`. Store processed IDs in a `processed_messages(id UUID PRIMARY KEY, processed_at TIMESTAMPTZ)` table and check before acting.

---

## 5. Read Replicas

### Gate — all must be true

- [ ] Read/write ratio exceeds 5:1 AND you have measured that the primary is the bottleneck
- [ ] Replication lag is acceptable for your read use cases (typically <100ms for sync replica)
- [ ] You have a plan for handling replication lag in code (stale reads, retry on not-found)

**Simpler alternative:** Add indexes. Run `EXPLAIN ANALYZE` on your slowest queries. PostgreSQL with correct indexes routinely handles 10K+ reads/sec on modest hardware. Measure before splitting.

### Go Implementation

```go
// internal/repository/db.go
package repository

import (
    "context"
    "fmt"
    "log/slog"

    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
)

// DBPool holds separate connections for writes and reads.
type DBPool struct {
    Primary  *sqlx.DB // reads + writes
    Replica  *sqlx.DB // reads only (can be nil — falls back to Primary)
}

// ReadDB returns the replica if available, otherwise the primary.
func (p *DBPool) ReadDB() *sqlx.DB {
    if p.Replica != nil {
        return p.Replica
    }
    return p.Primary
}

func NewDBPool(primaryDSN, replicaDSN string) (*DBPool, error) {
    primary, err := sqlx.Connect("postgres", primaryDSN)
    if err != nil {
        return nil, fmt.Errorf("connect primary: %w", err)
    }

    pool := &DBPool{Primary: primary}

    if replicaDSN != "" {
        replica, err := sqlx.Connect("postgres", replicaDSN)
        if err != nil {
            slog.Warn("replica unavailable, using primary for reads", "err", err)
        } else {
            pool.Replica = replica
        }
    }
    return pool, nil
}
```

**Usage in a repository:**

```go
type UserRepository struct {
    pool *repository.DBPool
}

func (r *UserRepository) FindByID(ctx context.Context, id string) (*User, error) {
    var u User
    err := r.pool.ReadDB().GetContext(ctx, &u, `SELECT * FROM users WHERE id = $1`, id)
    if err != nil {
        return nil, fmt.Errorf("find user: %w", err)
    }
    return &u, nil
}

func (r *UserRepository) Create(ctx context.Context, u *User) error {
    _, err := r.pool.Primary.NamedExecContext(ctx,
        `INSERT INTO users (id, email, name) VALUES (:id, :email, :name)`, u)
    if err != nil {
        return fmt.Errorf("create user: %w", err)
    }
    return nil
}
```

**Handling replication lag:**

```go
// After a write, read from primary for immediate consistency.
type contextKey string
const ReadYourWritesKey contextKey = "read_your_writes"

func (r *UserRepository) readDB(ctx context.Context) *sqlx.DB {
    if ryw, _ := ctx.Value(ReadYourWritesKey).(bool); ryw {
        return r.pool.Primary
    }
    return r.pool.ReadDB()
}
```

For PostgreSQL replication setup and monitoring: see [gin-psql-dba/references/replication-and-ha.md](../../gin-psql-dba/references/replication-and-ha.md).

---

## 6. Pattern Comparison Table

| Pattern | Complexity Cost | When Prerequisites Are Met | Simple Alternative |
|---|---|---|---|
| CQRS | HIGH (4/5) | Read/write shapes diverge + 10x read volume + replica not enough | Separate read/write methods + materialized view |
| Event Sourcing | VERY HIGH (5/5) | Compliance audit + temporal queries + multiple projections | `audit_log` table + regular CRUD |
| Saga (Orchestrator) | HIGH (4/5) | 3+ cross-service steps + compensations needed | Sequential calls with manual compensation |
| Transactional Outbox | MEDIUM (3/5) | Atomic DB + message publish required | Write DB then publish; retry on failure |
| Read Replicas | MEDIUM (3/5) | Read/write ratio > 5:1 AND primary is measured bottleneck | Add indexes; run EXPLAIN ANALYZE |

**Reading the cost scale:** 1 = one file, no new dependencies. 5 = new infrastructure, new failure modes, weeks of learning curve. Apply this cost to your current product stage before deciding.
