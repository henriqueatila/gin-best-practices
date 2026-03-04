# Cross-Cutting Concerns for Go Gin APIs

Observability, caching, security architecture, feature flags, and configuration management. Patterns that span all layers of a Gin application.

## Table of Contents

- [Observability](#observability)
  - [Structured Logging](#structured-logging)
  - [Metrics](#metrics)
  - [Distributed Tracing](#distributed-tracing)
  - [Health Checks](#health-checks)
- [Caching](#caching)
  - [HTTP Cache Headers](#http-cache-headers)
  - [In-Memory Cache](#in-memory-cache)
  - [Redis Cache](#redis-cache)
  - [Cache Invalidation](#cache-invalidation)
- [Security Architecture](#security-architecture)
  - [Defense in Depth](#defense-in-depth)
  - [Secret Management](#secret-management)
  - [Request Signing](#request-signing)
- [Feature Flags](#feature-flags)
- [Configuration Management](#configuration-management)
- [Graceful Degradation](#graceful-degradation)

---

## Observability

The three pillars: logs, metrics, traces. Start with logs (you already have them). Add metrics when you need dashboards. Add traces when debugging cross-service calls.

### Structured Logging

Use `log/slog` (stdlib since Go 1.21). JSON output in production, text in development.

```go
// Setup
func NewLogger(env string) *slog.Logger {
    var handler slog.Handler
    if env == "production" {
        handler = slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
            Level: slog.LevelInfo,
        })
    } else {
        handler = slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
            Level: slog.LevelDebug,
        })
    }
    return slog.New(handler)
}
```

**Request logging middleware:**

```go
func RequestLogger(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path
        query := c.Request.URL.RawQuery

        c.Next()

        latency := time.Since(start)
        status := c.Writer.Status()

        attrs := []slog.Attr{
            slog.String("method", c.Request.Method),
            slog.String("path", path),
            slog.String("query", query),
            slog.Int("status", status),
            slog.Duration("latency", latency),
            slog.String("ip", c.ClientIP()),
            slog.String("user_agent", c.Request.UserAgent()),
        }

        // Add request ID if present
        if reqID := c.GetHeader("X-Request-ID"); reqID != "" {
            attrs = append(attrs, slog.String("request_id", reqID))
        }

        // Add user ID if authenticated
        if userID, exists := c.Get("user_id"); exists {
            attrs = append(attrs, slog.Any("user_id", userID))
        }

        level := slog.LevelInfo
        if status >= 500 {
            level = slog.LevelError
        } else if status >= 400 {
            level = slog.LevelWarn
        }

        logger.LogAttrs(c.Request.Context(), level, "request", attrs...)
    }
}
```

**Context-aware logging in services:**

```go
func (s *OrderService) Create(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    s.logger.InfoContext(ctx, "creating order",
        "user_id", req.UserID,
        "item_count", len(req.Items),
    )

    order, err := s.repo.Create(ctx, req)
    if err != nil {
        s.logger.ErrorContext(ctx, "failed to create order",
            "user_id", req.UserID,
            "error", err,
        )
        return nil, fmt.Errorf("create order: %w", err)
    }

    return order, nil
}
```

### Metrics

Use `prometheus/client_golang` for metrics. Expose at `/metrics`.

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

// Define metrics
var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )

    dbQueryDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "db_query_duration_seconds",
            Help:    "Database query duration in seconds",
            Buckets: []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1},
        },
        []string{"query"},
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal, httpRequestDuration, dbQueryDuration)
}

// Metrics middleware
func MetricsMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        duration := time.Since(start).Seconds()
        status := strconv.Itoa(c.Writer.Status())

        httpRequestsTotal.WithLabelValues(c.Request.Method, c.FullPath(), status).Inc()
        httpRequestDuration.WithLabelValues(c.Request.Method, c.FullPath()).Observe(duration)
    }
}

// Expose metrics endpoint
r.GET("/metrics", gin.WrapH(promhttp.Handler()))
```

**Key metrics to track:**
- `http_requests_total` — request rate, error rate
- `http_request_duration_seconds` — latency percentiles (p50, p95, p99)
- `db_query_duration_seconds` — slow query detection
- `db_connections_active` — connection pool saturation
- Business metrics: `orders_created_total`, `payments_processed_total`

### Distributed Tracing

Use OpenTelemetry when you have 2+ services communicating.

```go
import (
    "go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/sdk/trace"
)

func initTracer(ctx context.Context, serviceName string) (*trace.TracerProvider, error) {
    exporter, err := otlptracehttp.New(ctx)
    if err != nil {
        return nil, fmt.Errorf("create exporter: %w", err)
    }

    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String(serviceName),
        )),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}

// Add to Gin
r.Use(otelgin.Middleware("myapp"))
```

**When to add tracing:**
- You have 2+ services calling each other
- You need to debug latency across service boundaries
- NOT for a monolith — structured logging is sufficient

### Health Checks

```go
type HealthChecker struct {
    db    *sqlx.DB
    redis *redis.Client // optional
}

func (h *HealthChecker) Liveness(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"status": "alive"})
}

func (h *HealthChecker) Readiness(c *gin.Context) {
    ctx, cancel := context.WithTimeout(c.Request.Context(), 2*time.Second)
    defer cancel()

    checks := gin.H{"status": "ready"}

    if err := h.db.PingContext(ctx); err != nil {
        checks["status"] = "not_ready"
        checks["db"] = err.Error()
        c.JSON(http.StatusServiceUnavailable, checks)
        return
    }
    checks["db"] = "ok"

    if h.redis != nil {
        if err := h.redis.Ping(ctx).Err(); err != nil {
            checks["status"] = "not_ready"
            checks["redis"] = err.Error()
            c.JSON(http.StatusServiceUnavailable, checks)
            return
        }
        checks["redis"] = "ok"
    }

    c.JSON(http.StatusOK, checks)
}

// Register
r.GET("/health/live", health.Liveness)
r.GET("/health/ready", health.Readiness)
```

---

## Caching

### Decision Tree

```
START: Is data read more than written?
  ├── No → No cache. Optimize queries instead (→ gin-psql-dba).
  └── Yes → Is data the same for all users?
      ├── Yes → HTTP Cache-Control headers. Zero infrastructure cost.
      └── No (per-user data) → Is total cached data < 100MB?
          ├── Yes → In-memory cache (ristretto, sync.Map).
          └── No → Redis. But first verify PostgreSQL + indexes isn't enough.
```

### HTTP Cache Headers

Cheapest caching — no server infrastructure needed.

```go
func CacheControl(maxAge time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Cache-Control", fmt.Sprintf("public, max-age=%d", int(maxAge.Seconds())))
        c.Next()
    }
}

// Static data: cache for 1 hour
r.GET("/api/v1/categories", CacheControl(1*time.Hour), handler.ListCategories)

// Dynamic data: no cache
r.GET("/api/v1/users/:id", handler.GetUser) // no cache middleware

// Conditional: ETag-based
func WithETag(c *gin.Context, data any) {
    body, _ := json.Marshal(data)
    etag := fmt.Sprintf(`"%x"`, sha256.Sum256(body))
    c.Header("ETag", etag)

    if c.GetHeader("If-None-Match") == etag {
        c.Status(http.StatusNotModified)
        return
    }

    c.Data(http.StatusOK, "application/json", body)
}
```

### In-Memory Cache

For small, frequently-read data. Use `dgraph-io/ristretto` for LRU with admission policy.

```go
import "github.com/dgraph-io/ristretto/v2"

func NewCache() (*ristretto.Cache[string, any], error) {
    return ristretto.NewCache(&ristretto.Config[string, any]{
        NumCounters: 1e5,     // 100K keys to track frequency
        MaxCost:     1 << 27, // 128MB max
        BufferItems: 64,
    })
}

// In service layer
func (s *CategoryService) List(ctx context.Context) ([]Category, error) {
    const cacheKey = "categories:all"
    if val, found := s.cache.Get(cacheKey); found {
        return val.([]Category), nil
    }

    categories, err := s.repo.ListAll(ctx)
    if err != nil {
        return nil, err
    }

    s.cache.SetWithTTL(cacheKey, categories, 1, 5*time.Minute)
    return categories, nil
}
```

### Redis Cache

For shared cache across multiple instances, large datasets, or TTL-based expiration.

```go
import "github.com/redis/go-redis/v9"

func (s *UserService) GetByID(ctx context.Context, id string) (*User, error) {
    cacheKey := "user:" + id

    // Try cache first
    cached, err := s.redis.Get(ctx, cacheKey).Bytes()
    if err == nil {
        var user User
        if err := json.Unmarshal(cached, &user); err == nil {
            return &user, nil
        }
    }

    // Cache miss — fetch from DB
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, err
    }

    // Cache for 10 minutes
    data, _ := json.Marshal(user)
    s.redis.Set(ctx, cacheKey, data, 10*time.Minute)

    return user, nil
}
```

### Cache Invalidation

**The hardest problem in computer science.** Keep it simple:

1. **TTL-based:** Set an expiration. Accept stale data for the TTL duration. Simplest.
2. **Write-through:** Update cache on every write. Consistent but adds write latency.
3. **Event-driven:** Invalidate on write events. Best consistency but most complex.

```go
// Write-through pattern
func (s *UserService) Update(ctx context.Context, id string, req UpdateUserRequest) (*User, error) {
    user, err := s.repo.Update(ctx, id, req)
    if err != nil {
        return nil, err
    }

    // Invalidate cache
    cacheKey := "user:" + id
    s.redis.Del(ctx, cacheKey)

    return user, nil
}
```

**Rule:** Start with TTL. Only add write-through if stale data causes real problems.

---

## Security Architecture

### Defense in Depth

Layer security — don't rely on a single control.

```
Internet → Load Balancer (TLS termination, DDoS protection)
         → Gin Middleware (rate limiting, CORS, auth)
           → Handler (input validation, sanitization)
             → Service (business rule enforcement, authorization)
               → Repository (parameterized queries, RLS)
                 → PostgreSQL (encryption at rest, network isolation)
```

Each layer reference:
- Load balancer / TLS → **gin-deploy** skill
- Rate limiting, CORS → **gin-api** skill (middleware reference)
- Auth middleware → **gin-auth** skill
- Input validation → **gin-api** skill
- Parameterized queries → **gin-database** skill
- RLS, encryption → **gin-psql-dba** skill

### Secret Management

```go
// 12-factor: secrets from environment variables
type Config struct {
    DatabaseURL string `env:"DATABASE_URL,required"`
    JWTSecret   string `env:"JWT_SECRET,required"`
    RedisURL    string `env:"REDIS_URL"`
    Port        string `env:"PORT" envDefault:"8080"`
}

// Use github.com/caarlos0/env/v11
func LoadConfig() (*Config, error) {
    cfg := &Config{}
    if err := env.Parse(cfg); err != nil {
        return nil, fmt.Errorf("parse config: %w", err)
    }
    return cfg, nil
}
```

**Rules:**
- Never hardcode secrets in code
- Never commit `.env` files to git
- Use `.env.example` with placeholder values
- In production: use Docker secrets, Kubernetes secrets, or a vault (HashiCorp, AWS SSM)
- Rotate secrets periodically — short-lived JWT tokens help

### Request Signing

For service-to-service auth when JWT is overkill:

```go
import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
)

func SignRequest(body []byte, secret string) string {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(body)
    return hex.EncodeToString(mac.Sum(nil))
}

func VerifySignature(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        signature := c.GetHeader("X-Signature")
        if signature == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "missing signature"})
            return
        }

        body, err := io.ReadAll(c.Request.Body)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{"error": "cannot read body"})
            return
        }
        c.Request.Body = io.NopCloser(bytes.NewBuffer(body))

        expected := SignRequest(body, secret)
        if !hmac.Equal([]byte(signature), []byte(expected)) {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid signature"})
            return
        }

        c.Next()
    }
}
```

---

## Feature Flags

**When you need them:** Gradual rollouts, A/B testing, kill switches for risky features. **When you don't:** Most features. Ship it or don't.

### Simple In-Code Flags (Start Here)

```go
type FeatureFlags struct {
    EnableNewCheckout bool `env:"FF_NEW_CHECKOUT" envDefault:"false"`
    EnableBetaSearch  bool `env:"FF_BETA_SEARCH"  envDefault:"false"`
}

func (h *OrderHandler) Checkout(c *gin.Context) {
    if h.flags.EnableNewCheckout {
        h.newCheckout(c)
        return
    }
    h.legacyCheckout(c)
}
```

### User-Targeted Flags (When You Outgrow Env Vars)

Use a feature flag service (LaunchDarkly, Unleash, Flipt) or a simple DB table:

```sql
CREATE TABLE feature_flags (
    name       TEXT PRIMARY KEY,
    enabled    BOOLEAN NOT NULL DEFAULT false,
    rules      JSONB,  -- {"user_ids": ["abc"], "percentage": 10}
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Rule:** Don't build your own feature flag system until env vars are clearly insufficient. Environment variables cover 90% of use cases.

---

## Configuration Management

### The 12-Factor Config Approach

All config from environment variables. Single `Config` struct loaded at startup.

```go
type Config struct {
    // Server
    Port         string `env:"PORT"          envDefault:"8080"`
    Environment  string `env:"ENVIRONMENT"   envDefault:"development"`

    // Database
    DatabaseURL  string `env:"DATABASE_URL,required"`
    MaxOpenConns int    `env:"DB_MAX_OPEN"   envDefault:"25"`

    // Auth
    JWTSecret    string `env:"JWT_SECRET,required"`
    TokenTTL     time.Duration `env:"TOKEN_TTL" envDefault:"15m"`

    // External
    PaymentAPIURL string `env:"PAYMENT_API_URL"`
    PaymentAPIKey string `env:"PAYMENT_API_KEY"`
}
```

**Rules:**
- One config struct, loaded once in `main.go`, passed via dependency injection
- Never read `os.Getenv` in service/repository code
- Use `envDefault` for development defaults
- Required fields fail fast at startup (not at first request)

---

## Graceful Degradation

When an external dependency fails, degrade gracefully instead of returning 500.

```go
func (s *ProductService) GetWithRecommendations(ctx context.Context, id string) (*ProductPage, error) {
    product, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get product: %w", err) // this is critical — propagate error
    }

    // Recommendations are non-critical — degrade gracefully
    recommendations, err := s.recommender.Get(ctx, id)
    if err != nil {
        s.logger.WarnContext(ctx, "recommendations unavailable, returning empty",
            "product_id", id,
            "error", err,
        )
        recommendations = []Product{} // empty, not nil
    }

    return &ProductPage{
        Product:         product,
        Recommendations: recommendations,
    }, nil
}
```

**Pattern:** Classify each dependency as **critical** (propagate error) or **non-critical** (degrade + log + return partial result).
