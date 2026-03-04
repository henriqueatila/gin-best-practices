# Messaging Patterns Reference

Practical guide to async messaging in Go Gin APIs using RabbitMQ. Covers when to go async, producer/consumer patterns, dead letter handling, and idempotency.

**Package:** `github.com/rabbitmq/amqp091-go`
**Logging:** `log/slog`
**Cost:** LOW → MEDIUM → HIGH (noted per section)

---

## Table of Contents

1. [When to Use Messaging](#1-when-to-use-messaging)
2. [RabbitMQ Setup](#2-rabbitmq-setup)
3. [Producer Pattern](#3-producer-pattern)
4. [Consumer Pattern](#4-consumer-pattern)
5. [Work Queues](#5-work-queues)
6. [Pub/Sub with Exchanges](#6-pubsub-with-exchanges)
7. [Dead Letter Queues](#7-dead-letter-queues)
8. [Idempotent Consumers](#8-idempotent-consumers)
9. [Docker Compose — RabbitMQ for Dev](#9-docker-compose--rabbitmq-for-dev)
10. [Cross-Skill References](#10-cross-skill-references)

---

## 1. When to Use Messaging

**Gate:** Add async messaging only when synchronous HTTP creates a user-facing bottleneck or reliability risk.

**Decision Tree:**

```
START: Does the caller need the result immediately?
  ├── Yes → Synchronous HTTP. Done.
  └── No → Can the work fail and be retried later?
      ├── No → Synchronous with timeout. Done.
      └── Yes → Message queue.
          └── Need exactly-once delivery?
              ├── No → Producer + consumer with manual ack.
              └── Yes → Transactional outbox + idempotent consumer.
```

**Good candidates for async:**
- Email / SMS notifications
- Image processing, PDF generation
- Webhook delivery to third parties
- Audit log writes
- Cache invalidation across services
- Batch data exports

**Bad candidates (keep sync):**
- Auth checks
- Payment confirmation (user waits)
- Read queries
- Anything requiring immediate response data

**Cost: LOW** — decision is free; wrong choice is expensive.

---

## 2. RabbitMQ Setup

**Gate:** One connection per process. One channel per goroutine. Never share channels across goroutines.

**Cost: LOW** — connection setup is one-time per service startup.

### Connection Factory with Reconnect

```go
// pkg/messaging/connection.go
package messaging

import (
	"context"
	"fmt"
	"log/slog"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

// Config holds RabbitMQ connection settings.
type Config struct {
	URL            string
	ReconnectDelay time.Duration
	MaxRetries     int
}

// Connection wraps an amqp.Connection with reconnect logic.
type Connection struct {
	cfg    Config
	conn   *amqp.Connection
	logger *slog.Logger
}

// NewConnection dials RabbitMQ and returns a managed connection.
func NewConnection(cfg Config, logger *slog.Logger) (*Connection, error) {
	if cfg.ReconnectDelay == 0 {
		cfg.ReconnectDelay = 5 * time.Second
	}
	if cfg.MaxRetries == 0 {
		cfg.MaxRetries = 10
	}

	c := &Connection{cfg: cfg, logger: logger}
	if err := c.dial(); err != nil {
		return nil, fmt.Errorf("messaging.NewConnection: %w", err)
	}
	return c, nil
}

func (c *Connection) dial() error {
	var err error
	for i := range c.cfg.MaxRetries {
		c.conn, err = amqp.Dial(c.cfg.URL)
		if err == nil {
			c.logger.Info("rabbitmq connected", "attempt", i+1)
			return nil
		}
		c.logger.Warn("rabbitmq dial failed, retrying",
			"attempt", i+1,
			"max", c.cfg.MaxRetries,
			"error", err,
			"delay", c.cfg.ReconnectDelay,
		)
		time.Sleep(c.cfg.ReconnectDelay)
	}
	return fmt.Errorf("rabbitmq dial: exhausted %d retries: %w", c.cfg.MaxRetries, err)
}

// Channel opens a new AMQP channel. Caller must close it.
func (c *Connection) Channel() (*amqp.Channel, error) {
	ch, err := c.conn.Channel()
	if err != nil {
		return nil, fmt.Errorf("messaging.Channel: %w", err)
	}
	return ch, nil
}

// Reconnect re-dials after connection loss. Call in a monitoring goroutine.
func (c *Connection) Reconnect(ctx context.Context) error {
	notifyClose := c.conn.NotifyClose(make(chan *amqp.Error, 1))
	select {
	case <-ctx.Done():
		return ctx.Err()
	case err := <-notifyClose:
		c.logger.Warn("rabbitmq connection closed, reconnecting", "error", err)
		return c.dial()
	}
}

// Close closes the underlying connection.
func (c *Connection) Close() error {
	return c.conn.Close()
}
```

### Queue Declaration Helper

```go
// pkg/messaging/topology.go
package messaging

import (
	"fmt"

	amqp "github.com/rabbitmq/amqp091-go"
)

// QueueOpts configures queue declaration.
type QueueOpts struct {
	Name       string
	Durable    bool
	AutoDelete bool
	// DLX is the dead letter exchange name (empty = no DLX).
	DLX string
}

// DeclareQueue declares a queue with optional dead letter exchange.
func DeclareQueue(ch *amqp.Channel, opts QueueOpts) (amqp.Queue, error) {
	args := amqp.Table{}
	if opts.DLX != "" {
		args["x-dead-letter-exchange"] = opts.DLX
	}

	q, err := ch.QueueDeclare(
		opts.Name,
		opts.Durable,
		opts.AutoDelete,
		false, // exclusive
		false, // no-wait
		args,
	)
	if err != nil {
		return amqp.Queue{}, fmt.Errorf("DeclareQueue %q: %w", opts.Name, err)
	}
	return q, nil
}
```

---

## 3. Producer Pattern

**Gate:** Publish from the service layer, never directly from Gin handlers. Keep handlers thin.

**Cost: LOW** — publishing is a fast network write; add only when you have real async work.

```go
// pkg/messaging/producer.go
package messaging

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
	"github.com/google/uuid"
)

// Producer publishes messages to RabbitMQ.
type Producer struct {
	conn   *Connection
	logger *slog.Logger
}

// NewProducer creates a Producer backed by conn.
func NewProducer(conn *Connection, logger *slog.Logger) *Producer {
	return &Producer{conn: conn, logger: logger}
}

// Publish serializes payload to JSON and publishes to exchange/routingKey.
// A unique message ID is set automatically for idempotency support.
func (p *Producer) Publish(ctx context.Context, exchange, routingKey string, payload any) error {
	body, err := json.Marshal(payload)
	if err != nil {
		return fmt.Errorf("producer.Publish marshal: %w", err)
	}

	ch, err := p.conn.Channel()
	if err != nil {
		return fmt.Errorf("producer.Publish channel: %w", err)
	}
	defer ch.Close()

	msgID := uuid.NewString()
	msg := amqp.Publishing{
		ContentType:  "application/json",
		DeliveryMode: amqp.Persistent, // survive broker restart
		MessageId:    msgID,
		Timestamp:    time.Now().UTC(),
		Body:         body,
	}

	if err := ch.PublishWithContext(ctx, exchange, routingKey, false, false, msg); err != nil {
		return fmt.Errorf("producer.Publish: %w", err)
	}

	p.logger.InfoContext(ctx, "message published",
		"exchange", exchange,
		"routing_key", routingKey,
		"message_id", msgID,
	)
	return nil
}
```

### Calling from a Service

```go
// internal/notification/service.go (excerpt)
func (s *Service) SendWelcomeEmail(ctx context.Context, userID string, email string) error {
	payload := map[string]string{
		"user_id": userID,
		"email":   email,
		"type":    "welcome",
	}
	if err := s.producer.Publish(ctx, "", "email.send", payload); err != nil {
		return fmt.Errorf("SendWelcomeEmail: %w", err)
	}
	return nil
}
```

---

## 4. Consumer Pattern

**Gate:** Run consumers as background goroutines in a dedicated worker binary or a separate goroutine started in `main.go`. Never block the HTTP server.

**Cost: MEDIUM** — each consumer goroutine holds a channel; size your channel pool accordingly.

```go
// internal/worker/email_consumer.go
package worker

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"

	amqp "github.com/rabbitmq/amqp091-go"
	"yourmodule/pkg/messaging"
)

// EmailConsumer processes email.send messages.
type EmailConsumer struct {
	conn    *messaging.Connection
	mailer  Mailer // your email abstraction
	logger  *slog.Logger
}

// EmailPayload is the expected message body shape.
type EmailPayload struct {
	UserID string `json:"user_id"`
	Email  string `json:"email"`
	Type   string `json:"type"`
}

// Run starts consuming. Blocks until ctx is cancelled.
func (c *EmailConsumer) Run(ctx context.Context) error {
	ch, err := c.conn.Channel()
	if err != nil {
		return fmt.Errorf("EmailConsumer.Run channel: %w", err)
	}
	defer ch.Close()

	// One unacked message at a time per consumer.
	if err := ch.Qos(1, 0, false); err != nil {
		return fmt.Errorf("EmailConsumer.Run qos: %w", err)
	}

	q, err := messaging.DeclareQueue(ch, messaging.QueueOpts{
		Name:    "email.send",
		Durable: true,
		DLX:     "dlx.email",
	})
	if err != nil {
		return fmt.Errorf("EmailConsumer.Run declare: %w", err)
	}

	deliveries, err := ch.Consume(q.Name, "", false, false, false, false, nil)
	if err != nil {
		return fmt.Errorf("EmailConsumer.Run consume: %w", err)
	}

	c.logger.Info("email consumer started", "queue", q.Name)

	for {
		select {
		case <-ctx.Done():
			c.logger.Info("email consumer shutting down")
			return nil
		case d, ok := <-deliveries:
			if !ok {
				return fmt.Errorf("EmailConsumer: delivery channel closed")
			}
			c.handle(ctx, d)
		}
	}
}

func (c *EmailConsumer) handle(ctx context.Context, d amqp.Delivery) {
	var payload EmailPayload
	if err := json.Unmarshal(d.Body, &payload); err != nil {
		c.logger.ErrorContext(ctx, "email consumer: bad payload, dead-lettering",
			"message_id", d.MessageId, "error", err)
		_ = d.Nack(false, false) // permanent failure → DLQ
		return
	}

	if err := c.mailer.Send(ctx, payload.Email, payload.Type); err != nil {
		c.logger.WarnContext(ctx, "email consumer: transient error, requeueing",
			"message_id", d.MessageId, "error", err)
		_ = d.Nack(false, true) // requeue for retry
		return
	}

	c.logger.InfoContext(ctx, "email sent", "message_id", d.MessageId, "to", payload.Email)
	_ = d.Ack(false)
}
```

---

## 5. Work Queues

**Gate:** Use competing consumers when a single consumer cannot keep up with throughput. Scale horizontally by running multiple instances of the same consumer.

**Cost: LOW** — same queue, multiple consumers; RabbitMQ round-robins by default.

```go
// cmd/worker/main.go
package main

import (
	"context"
	"log/slog"
	"os"
	"os/signal"
	"syscall"

	"yourmodule/internal/worker"
	"yourmodule/pkg/messaging"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	conn, err := messaging.NewConnection(messaging.Config{
		URL: os.Getenv("RABBITMQ_URL"),
	}, logger)
	if err != nil {
		logger.Error("failed to connect to rabbitmq", "error", err)
		os.Exit(1)
	}
	defer conn.Close()

	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	// Scale: run N=3 competing consumers on the same queue.
	for range 3 {
		consumer := worker.NewEmailConsumer(conn, logger)
		go func() {
			if err := consumer.Run(ctx); err != nil {
				logger.Error("email consumer error", "error", err)
			}
		}()
	}

	<-ctx.Done()
	logger.Info("worker shutting down")
}
```

**Key Rule:** Set `ch.Qos(1, 0, false)` so each consumer only prefetches one message. Without this, a busy consumer gets starved while another is overloaded.

---

## 6. Pub/Sub with Exchanges

**Gate:** Use exchanges when one event must fan out to multiple independent consumers (e.g., order.placed triggers inventory, email, and analytics).

**Cost: MEDIUM** — topology complexity grows; document your exchange/binding map.

### Fanout (Broadcast to All Queues)

```go
// pkg/messaging/pubsub.go
package messaging

import (
	"fmt"

	amqp "github.com/rabbitmq/amqp091-go"
)

// DeclareFanout declares a fanout exchange and binds a queue to it.
func DeclareFanout(ch *amqp.Channel, exchange, queueName string) error {
	if err := ch.ExchangeDeclare(exchange, "fanout", true, false, false, false, nil); err != nil {
		return fmt.Errorf("DeclareFanout exchange %q: %w", exchange, err)
	}
	q, err := DeclareQueue(ch, QueueOpts{Name: queueName, Durable: true})
	if err != nil {
		return err
	}
	if err := ch.QueueBind(q.Name, "", exchange, false, nil); err != nil {
		return fmt.Errorf("DeclareFanout bind %q→%q: %w", queueName, exchange, err)
	}
	return nil
}
```

### Topic Exchange (Pattern Routing)

```go
// Routing key examples: "order.placed", "order.cancelled", "user.created"
// Binding pattern:      "order.*"  matches order.placed and order.cancelled
//                       "#"        matches everything

func DeclareTopicBinding(ch *amqp.Channel, exchange, queueName, pattern string) error {
	if err := ch.ExchangeDeclare(exchange, "topic", true, false, false, false, nil); err != nil {
		return fmt.Errorf("DeclareTopicBinding exchange %q: %w", exchange, err)
	}
	q, err := DeclareQueue(ch, QueueOpts{Name: queueName, Durable: true})
	if err != nil {
		return err
	}
	if err := ch.QueueBind(q.Name, pattern, exchange, false, nil); err != nil {
		return fmt.Errorf("DeclareTopicBinding bind %q pattern=%q: %w", queueName, pattern, err)
	}
	return nil
}
```

### Example: Order Event Fan-out

```
Exchange: events (topic)
  ├── routing: "order.*"  → queue: inventory.orders  → InventoryConsumer
  ├── routing: "order.*"  → queue: email.orders      → EmailConsumer
  └── routing: "order.*"  → queue: analytics.orders  → AnalyticsConsumer
```

---

## 7. Dead Letter Queues

**Gate:** Add DLQ when message loss on failure is unacceptable. Without DLQ, rejected messages vanish.

**Cost: MEDIUM** — requires separate DLX, DLQ, and a monitoring consumer.

### Declare DLX and DLQ

```go
// pkg/messaging/dlq.go
package messaging

import (
	"fmt"

	amqp "github.com/rabbitmq/amqp091-go"
)

// SetupDLQ declares dead letter exchange + queue and binds them.
// Call this before declaring the primary queue that uses DLX.
func SetupDLQ(ch *amqp.Channel, dlxName, dlqName string) error {
	// Declare the dead letter exchange (direct type).
	if err := ch.ExchangeDeclare(dlxName, "direct", true, false, false, false, nil); err != nil {
		return fmt.Errorf("SetupDLQ exchange %q: %w", dlxName, err)
	}

	// Declare the dead letter queue.
	dlq, err := ch.QueueDeclare(dlqName, true, false, false, false, nil)
	if err != nil {
		return fmt.Errorf("SetupDLQ queue %q: %w", dlqName, err)
	}

	// Bind DLQ to DLX. Routing key "" catches all dead-lettered messages.
	if err := ch.QueueBind(dlq.Name, "", dlxName, false, nil); err != nil {
		return fmt.Errorf("SetupDLQ bind: %w", err)
	}

	return nil
}
```

### Primary Queue Using DLX

```go
// In consumer setup:
if err := messaging.SetupDLQ(ch, "dlx.email", "dlq.email"); err != nil {
    return fmt.Errorf("setup dlq: %w", err)
}

q, err := messaging.DeclareQueue(ch, messaging.QueueOpts{
    Name:    "email.send",
    Durable: true,
    DLX:     "dlx.email", // failed messages go here
})
```

### DLQ Monitor Consumer

```go
// internal/worker/dlq_monitor.go
package worker

import (
	"context"
	"log/slog"

	amqp "github.com/rabbitmq/amqp091-go"
	"yourmodule/pkg/messaging"
)

// DLQMonitor consumes from a dead letter queue and alerts.
type DLQMonitor struct {
	conn    *messaging.Connection
	alerter Alerter // PagerDuty, Slack, etc.
	logger  *slog.Logger
}

func (m *DLQMonitor) Run(ctx context.Context, dlqName string) error {
	ch, _ := m.conn.Channel()
	defer ch.Close()

	q, _ := messaging.DeclareQueue(ch, messaging.QueueOpts{Name: dlqName, Durable: true})
	deliveries, _ := ch.Consume(q.Name, "", false, false, false, false, nil)

	for {
		select {
		case <-ctx.Done():
			return nil
		case d, ok := <-deliveries:
			if !ok {
				return nil
			}
			m.logger.ErrorContext(ctx, "dead letter received",
				"message_id", d.MessageId,
				"routing_key", d.RoutingKey,
				"body", string(d.Body),
			)
			m.alerter.Alert(ctx, "dead letter: "+d.RoutingKey, d.Body)
			_ = d.Ack(false) // ack to remove from DLQ after alerting
		}
	}
}
```

---

## 8. Idempotent Consumers

**Gate:** Add idempotency when duplicate message delivery would cause real harm (double charge, duplicate record). RabbitMQ guarantees at-least-once delivery on requeue.

**Cost: MEDIUM** — requires Redis or PostgreSQL for dedup state.

### Redis-Based Dedup

```go
// internal/worker/idempotency.go
package worker

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

const dedupTTL = 24 * time.Hour

// Deduplicator checks and records processed message IDs.
type Deduplicator struct {
	rdb *redis.Client
}

// IsDuplicate returns true if messageID was already processed.
// Marks messageID as processed atomically on first call.
func (d *Deduplicator) IsDuplicate(ctx context.Context, messageID string) (bool, error) {
	key := "msg:processed:" + messageID
	// SET NX: only set if not exists. Returns 1 if set, 0 if already existed.
	ok, err := d.rdb.SetNX(ctx, key, "1", dedupTTL).Result()
	if err != nil {
		return false, fmt.Errorf("dedup check %q: %w", messageID, err)
	}
	return !ok, nil // !ok means key already existed → duplicate
}
```

### PostgreSQL-Based Dedup (for services without Redis)

```sql
-- migrations: create processed_messages table
CREATE TABLE processed_messages (
    message_id TEXT PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Optional: auto-clean old entries via pg_cron or a background job
```

```go
// PostgreSQL insert-or-ignore dedup check
func (d *PgDeduplicator) IsDuplicate(ctx context.Context, db *sql.DB, messageID string) (bool, error) {
	_, err := db.ExecContext(ctx,
		`INSERT INTO processed_messages (message_id) VALUES ($1) ON CONFLICT DO NOTHING`,
		messageID,
	)
	// If no rows affected, the message_id already existed → duplicate.
	// This is an approximation; use result.RowsAffected() for precision.
	if err != nil {
		return false, fmt.Errorf("pg dedup %q: %w", messageID, err)
	}
	return false, nil
}
```

### Usage in Consumer

```go
func (c *EmailConsumer) handle(ctx context.Context, d amqp.Delivery) {
	dup, err := c.dedup.IsDuplicate(ctx, d.MessageId)
	if err != nil {
		c.logger.WarnContext(ctx, "dedup check failed, processing anyway", "error", err)
	}
	if dup {
		c.logger.InfoContext(ctx, "duplicate message skipped", "message_id", d.MessageId)
		_ = d.Ack(false) // ack to remove from queue
		return
	}

	// ... proceed with normal processing
}
```

---

## 9. Docker Compose — RabbitMQ for Dev

**Gate:** Use this for local development only. In production, use a managed broker (CloudAMQP, AWS MQ) or a properly clustered RabbitMQ deployment.

**Cost: LOW** — single-node is fine for local dev; never run single-node in production without persistence config.

```yaml
# docker-compose.yml (RabbitMQ section)
services:
  rabbitmq:
    image: rabbitmq:4-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"    # AMQP protocol
      - "15672:15672"  # Management UI (http://localhost:15672)
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

volumes:
  rabbitmq_data:
```

**Environment variable for app:**

```bash
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
```

**Management UI credentials:** `guest / guest` at `http://localhost:15672`

**Useful management tasks via CLI inside container:**

```bash
# List queues
docker exec rabbitmq rabbitmqctl list_queues name messages consumers

# List exchanges
docker exec rabbitmq rabbitmqctl list_exchanges

# Purge a queue (dev only)
docker exec rabbitmq rabbitmqctl purge_queue email.send
```

---

## 10. Cross-Skill References

| Topic | Reference |
|---|---|
| Database transactions for outbox pattern | `data-patterns.md` → Transactional Outbox section |
| Circuit breaker around message publish | `resilience-patterns.md` → Circuit Breaker section |
| Deployment and scaling workers | `gin-deploy` skill → Worker Process section |
| Redis client setup for dedup | `data-patterns.md` → Redis section |
| Structured logging conventions | `gin-architect` skill → `logging-patterns.md` |

---

## Quick Reference

| Pattern | Use When | Client Call | Cost |
|---|---|---|---|
| Work queue | Task distribution across N workers | `ch.Consume` + `Qos(1)` | LOW |
| Fanout exchange | One event, many consumers | `ExchangeDeclare("fanout")` | MEDIUM |
| Topic exchange | Selective routing by key pattern | `ExchangeDeclare("topic")` | MEDIUM |
| DLQ | Message failure must not cause data loss | `x-dead-letter-exchange` arg | MEDIUM |
| Idempotent consumer | At-least-once delivery + side effects | Redis `SETNX` or PG `ON CONFLICT` | MEDIUM |
| Transactional outbox | Exactly-once across DB + broker | DB write + poller publishes | HIGH |

---

> **Note:** Always declare queues and exchanges idempotently (same arguments on every startup). RabbitMQ will error if you redeclare with different settings. Use `passive: false` on declare to assert existence.
