# Resilience Patterns Reference

Low-cost patterns for making Go Gin APIs resilient to external dependency failures. These are the patterns you can adopt freely — minimal complexity, high value.

**Default stance:** Add these to any service that calls external APIs. They cost little and prevent cascading failures.

## Table of Contents

1. [Circuit Breaker](#1-circuit-breaker)
2. [Bulkhead Pattern](#2-bulkhead-pattern)
3. [Retry with Exponential Backoff](#3-retry-with-exponential-backoff)
4. [Rate Limiting at Architecture Level](#4-rate-limiting-at-architecture-level)
5. [Pattern Comparison Table](#5-pattern-comparison-table)

---

## 1. Circuit Breaker

### Gate — you need this when

- [ ] Your service calls an external API or service that can be slow or unavailable
- [ ] Timeouts alone would block your goroutines for too long under a cascading failure

**Cost: LOW (2/5).** This is one of the few patterns you can add with minimal regret. A 50-line wrapper on your HTTP client is worth it for any external dependency.

**Simpler alternative:** A generous `http.Client` timeout + retry with backoff (see section 3). Add the circuit breaker once you observe that retries amplify load on the failing upstream.

### Go Implementation (`github.com/sony/gobreaker/v2`)

```go
// pkg/resilience/circuit_breaker.go
package resilience

import (
    "context"
    "fmt"
    "log/slog"
    "time"

    "github.com/sony/gobreaker/v2"
)

// NewCircuitBreaker returns a circuit breaker tuned for external HTTP calls.
// name: human-readable label for logs (e.g. "payment-api").
func NewCircuitBreaker(name string) *gobreaker.CircuitBreaker[[]byte] {
    return gobreaker.NewCircuitBreaker[[]byte](gobreaker.Settings{
        Name:        name,
        MaxRequests: 3,               // allow 3 requests in half-open state
        Interval:    60 * time.Second, // reset counts every 60s in closed state
        Timeout:     30 * time.Second, // wait 30s in open state before half-open
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            // Open after 5 consecutive failures.
            return counts.ConsecutiveFailures > 5
        },
        OnStateChange: func(name string, from, to gobreaker.State) {
            slog.Warn("circuit breaker state changed",
                "name", name, "from", from.String(), "to", to.String())
        },
    })
}

// CallWithBreaker wraps fn in a circuit breaker. Returns gobreaker.ErrOpenState when open.
func CallWithBreaker(ctx context.Context, cb *gobreaker.CircuitBreaker[[]byte], fn func() ([]byte, error)) ([]byte, error) {
    result, err := cb.Execute(func() ([]byte, error) {
        return fn()
    })
    if err != nil {
        return nil, fmt.Errorf("circuit breaker %s: %w", cb.Name(), err)
    }
    return result, nil
}
```

**Usage in a service:**

```go
// internal/payment/client.go
package payment

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "time"

    "github.com/sony/gobreaker/v2"
    "myapp/pkg/resilience"
)

type Client struct {
    http *http.Client
    cb   *gobreaker.CircuitBreaker[[]byte]
}

func NewClient() *Client {
    return &Client{
        http: &http.Client{Timeout: 5 * time.Second},
        cb:   resilience.NewCircuitBreaker("payment-api"),
    }
}

func (c *Client) Charge(ctx context.Context, orderID string) error {
    body, err := resilience.CallWithBreaker(ctx, c.cb, func() ([]byte, error) {
        req, _ := http.NewRequestWithContext(ctx, "POST", "https://payment.example.com/charge", nil)
        resp, err := c.http.Do(req)
        if err != nil {
            return nil, err
        }
        defer resp.Body.Close()
        if resp.StatusCode >= 500 {
            return nil, fmt.Errorf("upstream error: %d", resp.StatusCode)
        }
        return io.ReadAll(resp.Body)
    })
    _ = body
    return err
}
```

---

## 2. Bulkhead Pattern

### Gate — you need this when

- [ ] One slow dependency (database, external API) is causing goroutine saturation that affects unrelated parts of your service
- [ ] You can clearly identify 2+ distinct dependency groups with different SLA expectations

**Cost: LOW (2/5).** A semaphore is 5 lines of Go.

**Simpler alternative:** Set an aggressive `context.WithTimeout` on the slow call. This prevents indefinite blocking without building a separate pool. Add the bulkhead when you need concurrent capacity control, not just timeout protection.

### Go Implementation

```go
// pkg/resilience/bulkhead.go
package resilience

import (
    "context"
    "fmt"
)

// Bulkhead limits concurrent executions via a semaphore channel.
type Bulkhead struct {
    name string
    sem  chan struct{}
}

// NewBulkhead creates a bulkhead that allows at most maxConcurrent simultaneous calls.
func NewBulkhead(name string, maxConcurrent int) *Bulkhead {
    return &Bulkhead{
        name: name,
        sem:  make(chan struct{}, maxConcurrent),
    }
}

// Execute runs fn if a slot is available. Returns context.Err if ctx is cancelled while waiting.
func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
        return fn()
    case <-ctx.Done():
        return fmt.Errorf("bulkhead %s: %w", b.name, ctx.Err())
    }
}
```

**Usage — isolate database calls from external API calls:**

```go
// internal/app/app.go
var (
    dbBulkhead  = resilience.NewBulkhead("database", 50)
    apiBulkhead = resilience.NewBulkhead("payment-api", 10)
)

func (s *OrderService) CreateOrder(ctx context.Context, cmd order.CreateOrderCommand) (string, error) {
    var id string
    err := dbBulkhead.Execute(ctx, func() error {
        var e error
        id, e = s.repo.Create(ctx, cmd)
        return e
    })
    return id, err
}
```

---

## 3. Retry with Exponential Backoff

### Gate — always use when calling external services

**Cost: LOW (1/5).** Apply by default to any external HTTP or gRPC call. Not needed for local database calls (the driver handles reconnects; adding retries can cause double-writes).

**Jitter** prevents retry storms when many goroutines fail simultaneously.

### Go Implementation

```go
// pkg/resilience/retry.go
package resilience

import (
    "context"
    "fmt"
    "math/rand"
    "time"
)

// RetryConfig controls retry behaviour.
type RetryConfig struct {
    MaxRetries  int           // number of retries (not total attempts)
    BaseBackoff time.Duration // base sleep before first retry (doubles each attempt)
    MaxBackoff  time.Duration // cap on sleep duration
}

// DefaultRetryConfig is a sensible starting point for external HTTP calls.
var DefaultRetryConfig = RetryConfig{
    MaxRetries:  3,
    BaseBackoff: 100 * time.Millisecond,
    MaxBackoff:  2 * time.Second,
}

// Retry calls fn up to cfg.MaxRetries+1 times with exponential backoff and jitter.
func Retry(ctx context.Context, cfg RetryConfig, fn func() error) error {
    var lastErr error
    for attempt := 0; attempt <= cfg.MaxRetries; attempt++ {
        if err := fn(); err != nil {
            lastErr = err
            if attempt == cfg.MaxRetries {
                break
            }
            backoff := cfg.BaseBackoff * (1 << uint(attempt))
            if backoff > cfg.MaxBackoff {
                backoff = cfg.MaxBackoff
            }
            jitter := time.Duration(rand.Int63n(int64(backoff / 2)))
            select {
            case <-time.After(backoff + jitter):
            case <-ctx.Done():
                return fmt.Errorf("retry cancelled: %w", ctx.Err())
            }
            continue
        }
        return nil
    }
    return fmt.Errorf("after %d retries: %w", cfg.MaxRetries, lastErr)
}
```

**Usage:**

```go
err := resilience.Retry(ctx, resilience.DefaultRetryConfig, func() error {
    return paymentClient.Charge(ctx, orderID)
})
```

**When NOT to retry:** database writes (risk double-insert), non-idempotent operations without idempotency keys, errors that are clearly permanent (4xx client errors).

---

## 4. Rate Limiting at Architecture Level

Rate limiting in a single Gin instance (middleware) is covered in [gin-api/references/rate-limiting.md](../../gin-api/references/rate-limiting.md). This section covers architectural decisions.

### Decision Matrix

| Scenario | Approach |
|---|---|
| Single binary, simple protection | In-memory token bucket (no Redis) |
| Multiple instances, IP-based limits | Redis token bucket with Lua |
| Per-user billing accuracy | Redis sliding window, key = user ID |
| Global API quota across all routes | API gateway (Kong, Traefik, nginx) |
| Rate limiting for paid tiers | Redis + tier config loaded from env |

### Per-Service Limits

Each internal service should define its own limits independently. Do not use a shared rate limiter across services — failure domains must stay isolated.

```go
// Per-route group limits — different SLAs for different endpoints
public := r.Group("/api/v1")
public.Use(middleware.MemoryRateLimiter(middleware.MemoryRateLimiterConfig{
    Rate: 10, Burst: 20,
}, done))

authenticated := r.Group("/api/v1")
authenticated.Use(authMiddleware, middleware.TieredRateLimiter(rdb, tiers, "rl:auth:"))
```

### Distributed Rate Limiting with Redis

For multi-instance deployments, all instances share the same Redis counter. See `gin-api/references/rate-limiting.md` for the Lua-based token bucket and sliding window implementations.

**Key design choices:**
- Key by user ID (from JWT `sub` claim) over IP — survives NAT and mobile IP changes
- Fail open on Redis errors — blocking all traffic because a cache is down is worse than briefly allowing excess
- Set `X-RateLimit-Remaining` on every response, not only on 429

---

## 5. Pattern Comparison Table

| Pattern | Complexity Cost | When to Use | Simple Alternative |
|---|---|---|---|
| Circuit Breaker | LOW (2/5) | Any external dependency that can be slow/down | Timeout + retry backoff |
| Bulkhead | LOW (2/5) | One slow dep saturating goroutines affecting others | `context.WithTimeout` on slow calls |
| Retry + Backoff | LOW (1/5) | Any external call (default: always use) | Single call with timeout |
| Rate Limiting (distributed) | MEDIUM (3/5) | Multiple instances + per-user accuracy | In-memory token bucket (single node) |

**Reading the cost scale:** 1 = one file, no new dependencies. 5 = new infrastructure, new failure modes, weeks of learning curve.
