# Observability Reference

This file covers OpenTelemetry (OTel) instrumentation for Go Gin APIs: distributed tracing, RED metrics, and log-trace correlation. Use when adding observability to a Gin service, configuring an OTLP exporter, or setting up a local Jaeger stack. Covers the full pipeline from SDK initialization to graceful shutdown.

> **Architectural recommendation:** OTel Go SDK is stable at v1.x. Pin your `go.opentelemetry.io/otel` and `go.opentelemetry.io/contrib` versions explicitly — contrib packages release independently and may break without a pin.

## Table of Contents

1. [OTel SDK Setup](#otel-sdk-setup)
2. [Gin Middleware](#gin-middleware)
3. [Manual Spans in Service Layer](#manual-spans-in-service-layer)
4. [Error Recording](#error-recording)
5. [Metrics Middleware](#metrics-middleware)
6. [Log-Trace Correlation](#log-trace-correlation)
7. [Sampling](#sampling)
8. [Docker Compose — Local Observability Stack](#docker-compose--local-observability-stack)
9. [Graceful Shutdown](#graceful-shutdown)

---

## OTel SDK Setup

The OTel SDK uses a provider → exporter → processor pipeline. Initialize both `TracerProvider` and `MeterProvider` once at startup and register them as globals.

**Why OTLP gRPC:** The OTLP protocol is the vendor-neutral standard. It works with Jaeger, Grafana Tempo, Honeycomb, Datadog, and any OTel Collector. gRPC is preferred over HTTP for lower overhead in high-throughput services.

```go
// internal/telemetry/telemetry.go
package telemetry

import (
	"context"
	"fmt"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
	"go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc"
	"go.opentelemetry.io/otel/propagation"
	sdkmetric "go.opentelemetry.io/otel/sdk/metric"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

// Providers holds the initialized providers for graceful shutdown.
type Providers struct {
	TracerProvider *sdktrace.TracerProvider
	MeterProvider  *sdkmetric.MeterProvider
}

// Init initializes OTel tracing and metrics, registers globals, and returns
// providers for shutdown. otlpEndpoint is host:port (e.g. "localhost:4317").
func Init(ctx context.Context, serviceName, serviceVersion, otlpEndpoint string) (*Providers, error) {
	res, err := resource.New(ctx,
		resource.WithAttributes(
			semconv.ServiceName(serviceName),
			semconv.ServiceVersion(serviceVersion),
		),
		resource.WithFromEnv(),   // picks up OTEL_RESOURCE_ATTRIBUTES
		resource.WithProcess(),   // pid, executable name
		resource.WithOS(),
	)
	if err != nil {
		return nil, fmt.Errorf("telemetry: create resource: %w", err)
	}

	conn, err := grpc.NewClient(otlpEndpoint,
		grpc.WithTransportCredentials(insecure.NewCredentials()), // use TLS in production
	)
	if err != nil {
		return nil, fmt.Errorf("telemetry: dial otlp endpoint %s: %w", otlpEndpoint, err)
	}

	// ── Tracing ──────────────────────────────────────────────────────────────
	traceExporter, err := otlptracegrpc.New(ctx, otlptracegrpc.WithGRPCConn(conn))
	if err != nil {
		return nil, fmt.Errorf("telemetry: create trace exporter: %w", err)
	}

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(traceExporter),
		sdktrace.WithResource(res),
		sdktrace.WithSampler(productionSampler()),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{}, // W3C traceparent header
		propagation.Baggage{},
	))

	// ── Metrics ──────────────────────────────────────────────────────────────
	metricExporter, err := otlpmetricgrpc.New(ctx, otlpmetricgrpc.WithGRPCConn(conn))
	if err != nil {
		return nil, fmt.Errorf("telemetry: create metric exporter: %w", err)
	}

	mp := sdkmetric.NewMeterProvider(
		sdkmetric.WithReader(sdkmetric.NewPeriodicReader(metricExporter,
			sdkmetric.WithInterval(15*time.Second),
		)),
		sdkmetric.WithResource(res),
	)
	otel.SetMeterProvider(mp)

	return &Providers{TracerProvider: tp, MeterProvider: mp}, nil
}

func productionSampler() sdktrace.Sampler {
	// See Sampling section for full explanation.
	return sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))
}
```

**go.mod dependencies:**

```
go.opentelemetry.io/otel v1.33.0
go.opentelemetry.io/otel/sdk v1.33.0
go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc v1.33.0
go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc v1.33.0
go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin v0.58.0
google.golang.org/grpc v1.69.0
```

---

## Gin Middleware

`otelgin.Middleware` automatically creates a span for every HTTP request, sets standard HTTP attributes (`http.method`, `http.route`, `http.status_code`), and propagates W3C `traceparent` headers from incoming requests.

**Why register before other middleware:** The span wraps the full request lifecycle. Register `otelgin` first so that any subsequent middleware (auth, logging) runs inside the span.

```go
// cmd/api/main.go
package main

import (
	"log/slog"
	"net/http"
	"os"

	"github.com/gin-gonic/gin"
	"go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
)

func setupRouter(serviceName string, logger *slog.Logger) *gin.Engine {
	r := gin.New()

	// OTel tracing — must be first to wrap entire request lifecycle
	r.Use(otelgin.Middleware(serviceName))

	// Recovery and logging after otelgin so they run within the trace span
	r.Use(middleware.Recovery(logger))
	r.Use(requestLogger(logger))

	return r
}

func requestLogger(logger *slog.Logger) gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Next()
		logger.InfoContext(c.Request.Context(), "request",
			"method", c.Request.Method,
			"path",   c.Request.URL.Path,
			"status", c.Writer.Status(),
		)
	}
}
```

**Route group example:**

```go
v1 := r.Group("/api/v1")
{
	v1.GET("/users/:id", userHandler.GetByID)
	v1.POST("/users", userHandler.Create)
}
```

`otelgin` uses the registered route pattern (`/users/:id`) as the span name, not the resolved path — preventing high-cardinality spans.

---

## Manual Spans in Service Layer

Automatic middleware creates one span per HTTP request. Create child spans for significant operations in the service and repository layers to get call-level visibility.

**Why:** Without manual spans, you see "POST /orders took 400ms" but not whether it was the DB query, an external HTTP call, or business logic that was slow.

```go
// internal/service/order_service.go
package service

import (
	"context"
	"fmt"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/trace"
)

// tracer is package-level; name matches the instrumentation scope.
var tracer = otel.Tracer("myapp/service/order")

type OrderService struct {
	repo OrderRepository
}

func (s *OrderService) CreateOrder(ctx context.Context, userID string, items []Item) (*Order, error) {
	ctx, span := tracer.Start(ctx, "OrderService.CreateOrder",
		trace.WithAttributes(
			attribute.String("order.user_id", userID),
			attribute.Int("order.item_count", len(items)),
		),
	)
	defer span.End()

	order, err := s.repo.Insert(ctx, userID, items) // ctx carries the span
	if err != nil {
		return nil, fmt.Errorf("create order: %w", err)
	}

	span.SetAttributes(attribute.String("order.id", order.ID))
	return order, nil
}
```

**Key rules:**
- Always `defer span.End()` immediately after `tracer.Start` — never forget to close a span
- Pass `ctx` (not `c.Request.Context()` — that's only in handlers) down the call chain
- Name spans as `"Package.Method"` for easy filtering in Jaeger

---

## Error Recording

A span's status is `Unset` by default — not OK, not Error. Set it explicitly so trace UIs can filter on failed spans.

```go
// internal/service/order_service.go
import (
	"context"
	"fmt"

	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/trace"
)

func (s *OrderService) CreateOrder(ctx context.Context, userID string, items []Item) (*Order, error) {
	ctx, span := tracer.Start(ctx, "OrderService.CreateOrder")
	defer span.End()

	order, err := s.repo.Insert(ctx, userID, items)
	if err != nil {
		// RecordError attaches the error as a span event with stack trace.
		span.RecordError(err)
		// SetStatus marks the span as failed — visible in Jaeger's error filter.
		span.SetStatus(codes.Error, err.Error())
		return nil, fmt.Errorf("create order: %w", err)
	}

	span.SetStatus(codes.Ok, "")
	return order, nil
}
```

**RecordError vs SetStatus:**
- `span.RecordError(err)` — attaches error details as a structured span event (message, stack)
- `span.SetStatus(codes.Error, msg)` — marks the overall span status; used by trace UIs for error highlighting and alerting

Always call both. `RecordError` alone does not mark the span as failed in the UI.

---

## Metrics Middleware

OTel metrics expose RED metrics (Rate, Errors, Duration) for every route. Use the global `MeterProvider` (registered in `Init`) to create instruments once at startup.

**Why custom middleware over otelgin metrics:** `otelgin` focuses on tracing. Custom metric middleware gives control over metric names, label cardinality, and histogram bucket boundaries.

```go
// internal/middleware/metrics_middleware.go
package middleware

import (
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/metric"
)

type HTTPMetrics struct {
	requestDuration metric.Float64Histogram
	requestsTotal   metric.Int64Counter
}

// NewHTTPMetrics creates OTel metric instruments. Call once at startup.
func NewHTTPMetrics() (*HTTPMetrics, error) {
	meter := otel.Meter("myapp/http")

	duration, err := meter.Float64Histogram(
		"http.server.request.duration",
		metric.WithDescription("HTTP request duration in seconds"),
		metric.WithUnit("s"),
		metric.WithExplicitBucketBoundaries(
			0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10,
		),
	)
	if err != nil {
		return nil, err
	}

	total, err := meter.Int64Counter(
		"http.server.requests.total",
		metric.WithDescription("Total HTTP requests"),
	)
	if err != nil {
		return nil, err
	}

	return &HTTPMetrics{requestDuration: duration, requestsTotal: total}, nil
}

// Handler returns a Gin middleware that records request metrics.
func (m *HTTPMetrics) Handler() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()

		c.Next()

		attrs := []attribute.KeyValue{
			attribute.String("http.method", c.Request.Method),
			attribute.String("http.route", c.FullPath()), // pattern, not resolved path
			attribute.String("http.status_code", strconv.Itoa(c.Writer.Status())),
		}

		m.requestDuration.Record(c.Request.Context(),
			time.Since(start).Seconds(),
			metric.WithAttributes(attrs...),
		)
		m.requestsTotal.Add(c.Request.Context(), 1,
			metric.WithAttributes(attrs...),
		)
	}
}
```

Register in `main.go`:

```go
httpMetrics, err := middleware.NewHTTPMetrics()
if err != nil {
	logger.Error("failed to create http metrics", "error", err)
	os.Exit(1)
}

r := setupRouter(cfg.ServiceName, logger)
r.Use(httpMetrics.Handler())
```

**Cardinality warning:** Never use `c.Request.URL.Path` (resolved path) as a label — it creates unbounded cardinality (one label value per unique ID). Always use `c.FullPath()` (route pattern like `/users/:id`).

---

## Log-Trace Correlation

Connecting logs to traces lets you jump from a log line to the full trace in Jaeger. Inject `trace_id` and `span_id` into every log record produced within a request.

**Why slog:** `log/slog` supports structured key-value logging and a composable `Handler` interface, making trace injection clean without third-party libraries.

```go
// internal/telemetry/trace_log_handler.go
package telemetry

import (
	"context"
	"log/slog"

	"go.opentelemetry.io/otel/trace"
)

// TraceLogHandler wraps any slog.Handler and injects trace_id/span_id
// from the context into every log record.
type TraceLogHandler struct {
	inner slog.Handler
}

func NewTraceLogHandler(inner slog.Handler) *TraceLogHandler {
	return &TraceLogHandler{inner: inner}
}

func (h *TraceLogHandler) Enabled(ctx context.Context, level slog.Level) bool {
	return h.inner.Enabled(ctx, level)
}

func (h *TraceLogHandler) Handle(ctx context.Context, r slog.Record) error {
	if span := trace.SpanFromContext(ctx); span.SpanContext().IsValid() {
		sc := span.SpanContext()
		r.AddAttrs(
			slog.String("trace_id", sc.TraceID().String()),
			slog.String("span_id", sc.SpanID().String()),
		)
	}
	return h.inner.Handle(ctx, r)
}

func (h *TraceLogHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	return &TraceLogHandler{inner: h.inner.WithAttrs(attrs)}
}

func (h *TraceLogHandler) WithGroup(name string) slog.Handler {
	return &TraceLogHandler{inner: h.inner.WithGroup(name)}
}
```

Initialize in `main.go`:

```go
baseHandler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo})
logger := slog.New(telemetry.NewTraceLogHandler(baseHandler))
slog.SetDefault(logger)
```

Log within a handler using `c.Request.Context()`:

```go
func (h *UserHandler) GetByID(c *gin.Context) {
	user, err := h.svc.GetUser(c.Request.Context(), c.Param("id"))
	if err != nil {
		// trace_id and span_id are injected automatically
		h.logger.ErrorContext(c.Request.Context(), "get user failed",
			"user_id", c.Param("id"),
			"error", err,
		)
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}
	c.JSON(http.StatusOK, user)
}
```

Example log output (JSON):

```json
{
  "time": "2024-11-01T10:00:00Z",
  "level": "ERROR",
  "msg": "get user failed",
  "user_id": "123",
  "error": "sql: no rows in result set",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7"
}
```

---

## Sampling

Sampling controls what fraction of traces are exported. Sampling 100% of traces in production is expensive — a high-traffic API can generate gigabytes of trace data per hour.

**Strategy:** Use `ParentBased(TraceIDRatioBased)` so that:
- If an upstream service already started a trace (W3C `traceparent` header present), respect its sampling decision — don't drop a trace mid-flight
- For root spans (no parent), sample at a configured ratio

```go
// internal/telemetry/sampler.go
package telemetry

import (
	"os"
	"strconv"

	sdktrace "go.opentelemetry.io/otel/sdk/trace"
)

// SamplerFromEnv reads OTEL_SAMPLING_RATE (0.0–1.0).
// Defaults to 1.0 (100%) when unset — good for development.
// Set to 0.1 in production for 10% sampling.
func SamplerFromEnv() sdktrace.Sampler {
	rate := 1.0
	if v := os.Getenv("OTEL_SAMPLING_RATE"); v != "" {
		if parsed, err := strconv.ParseFloat(v, 64); err == nil {
			rate = parsed
		}
	}
	return sdktrace.ParentBased(sdktrace.TraceIDRatioBased(rate))
}
```

Use in `Init`:

```go
tp := sdktrace.NewTracerProvider(
	sdktrace.WithBatcher(traceExporter),
	sdktrace.WithResource(res),
	sdktrace.WithSampler(SamplerFromEnv()),
)
```

| Environment | `OTEL_SAMPLING_RATE` | Rationale |
|-------------|----------------------|-----------|
| Development | `1.0` (default) | See every trace while debugging |
| Staging | `0.5` | 50% — catch issues without full cost |
| Production | `0.1` | 10% — statistically representative |

**Head-based vs tail-based:** `TraceIDRatioBased` is head-based (decision made at trace start). For tail-based sampling (keep all error traces, sample successful ones), use an OTel Collector with the `tail_sampling` processor.

---

## Docker Compose — Local Observability Stack

Add Jaeger (all-in-one) and an OTel Collector to the existing docker-compose. The Collector receives OTLP from the app and forwards to Jaeger.

```yaml
# docker-compose.observability.yml
# Run alongside the main docker-compose.yml:
#   docker compose -f docker-compose.yml -f docker-compose.observability.yml up

services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.117.0
    command: ["--config=/etc/otel/config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel/config.yaml:ro
    ports:
      - "4317:4317"   # OTLP gRPC — app sends traces/metrics here
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Collector self-metrics (Prometheus scrape)
    depends_on:
      - jaeger

  jaeger:
    image: jaegertracing/all-in-one:1.65
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    ports:
      - "16686:16686"  # Jaeger UI
      - "4317"         # OTLP gRPC — internal only (used by collector)
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:16686"]
      interval: 10s
      timeout: 5s
      retries: 5
```

OTel Collector config:

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 512

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger]
```

App environment variables:

```yaml
# in docker-compose.yml app service
environment:
  - OTLP_ENDPOINT=otel-collector:4317
  - OTEL_SAMPLING_RATE=1.0   # 100% in local dev
```

Access Jaeger UI at `http://localhost:16686` after `docker compose up`.

---

## Prometheus + Grafana Stack

The existing OTel Collector pipeline exports traces to Jaeger. To also scrape metrics into Prometheus and visualise RED (Rate, Errors, Duration) in Grafana, add a `prometheus` exporter to the collector and bring up Prometheus + Grafana as a compose overlay.

### Updated OTel Collector config

Add the `prometheus` exporter to `otel-collector-config.yml` (already used in the base compose):

```yaml
# otel-collector-config.yml  — diff from existing file
exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
  # Expose scraped app metrics on :8889 so Prometheus can pull them
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: gin_app          # prefix: gin_app_http_request_duration_seconds …

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger, prometheus]   # added prometheus here
```

Port 8889 is the app-metrics endpoint; port 8888 remains the collector's own self-metrics.

### Compose overlay: `docker-compose.monitoring.yml`

```yaml
# docker-compose.monitoring.yml
# Usage: docker compose -f docker-compose.yml -f docker-compose.monitoring.yml up
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"
    depends_on:
      otel-collector:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:9090/-/ready"]
      interval: 10s
      timeout: 5s
      retries: 5

  grafana:
    image: grafana/grafana:latest
    environment:
      # Anonymous access is fine for local dev — remove in production
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    ports:
      - "3000:3000"
    depends_on:
      prometheus:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/api/health"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### `prometheus.yml` scrape config

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  # Collector self-metrics (queue sizes, export errors, etc.)
  - job_name: otel-collector
    static_configs:
      - targets: ["otel-collector:8888"]

  # App metrics forwarded by the prometheus exporter in the collector
  - job_name: gin-app
    static_configs:
      - targets: ["otel-collector:8889"]
```

### Grafana datasource provisioning

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    # Mark as default so new dashboards pick it up automatically
    isDefault: true
    editable: false
```

### Grafana dashboard provisioning (RED metrics)

```yaml
# grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1
providers:
  - name: default
    folder: Gin App
    type: file
    options:
      path: /etc/grafana/provisioning/dashboards
```

Save the dashboard JSON below as `grafana/provisioning/dashboards/gin-red.json`. It references the three metrics emitted by the existing `MetricsMiddleware`:

```json
{
  "title": "Gin RED Metrics",
  "uid": "gin-red-001",
  "schemaVersion": 38,
  "panels": [
    {
      "type": "stat",
      "title": "Request Rate (req/s)",
      "gridPos": {"x": 0, "y": 0, "w": 8, "h": 4},
      "targets": [{
        "expr": "rate(gin_app_http_request_duration_seconds_count[1m])",
        "legendFormat": "{{route}}"
      }]
    },
    {
      "type": "stat",
      "title": "Error Rate (%)",
      "gridPos": {"x": 8, "y": 0, "w": 8, "h": 4},
      "targets": [{
        "expr": "rate(gin_app_http_requests_total{status_code=~\"5..\"}[1m]) / rate(gin_app_http_requests_total[1m]) * 100",
        "legendFormat": "{{route}}"
      }]
    },
    {
      "type": "graph",
      "title": "P99 Latency (s)",
      "gridPos": {"x": 0, "y": 4, "w": 24, "h": 8},
      "targets": [{
        "expr": "histogram_quantile(0.99, rate(gin_app_http_request_duration_seconds_bucket[5m]))",
        "legendFormat": "p99 {{route}}"
      }]
    }
  ]
}
```

### Running the full stack

```bash
# Start everything: app + Jaeger + OTel Collector + Prometheus + Grafana
docker compose -f docker-compose.yml -f docker-compose.monitoring.yml up

# Verify endpoints
# Jaeger UI:       http://localhost:16686
# Prometheus UI:   http://localhost:9090
# Grafana:         http://localhost:3000  (anonymous viewer)
# OTel metrics:    http://localhost:8888/metrics  (collector self)
# App metrics:     http://localhost:8889/metrics  (gin_app_* series)
```

---

## Graceful Shutdown

OTel providers buffer telemetry in memory and flush to the exporter in batches. Skipping shutdown on process exit drops the last batch of spans and metrics — losing visibility into the final seconds before shutdown.

**Why this matters:** Kubernetes sends `SIGTERM`, waits (default 30s), then `SIGKILL`. Proper shutdown ensures all buffered traces from in-flight requests are exported before the process exits.

```go
// cmd/api/main.go
package main

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
	"go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
	"myapp/internal/config"
	"myapp/internal/telemetry"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	cfg, err := config.Load()
	if err != nil {
		logger.Error("config load failed", "error", err)
		os.Exit(1)
	}

	// Initialize OTel — must happen before any tracer/meter is used
	ctx := context.Background()
	providers, err := telemetry.Init(ctx, cfg.ServiceName, cfg.Version, cfg.OTLPEndpoint)
	if err != nil {
		logger.Error("telemetry init failed", "error", err)
		os.Exit(1)
	}

	r := gin.New()
	r.Use(otelgin.Middleware(cfg.ServiceName))
	// ... register routes

	srv := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           r,
		ReadHeaderTimeout: 5 * time.Second,  // Slowloris protection
		IdleTimeout:       120 * time.Second,
	}

	// Start server in background
	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			logger.Error("server error", "error", err)
			os.Exit(1)
		}
	}()

	logger.Info("server started", "port", cfg.Port)

	// Wait for OS signal
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	logger.Info("shutting down")

	// Give in-flight requests time to complete
	shutdownCtx, cancel := context.WithTimeout(context.Background(), cfg.ShutdownTimeout)
	defer cancel()

	if err := srv.Shutdown(shutdownCtx); err != nil {
		logger.Error("server shutdown error", "error", err)
	}

	// Flush and close OTel providers — order matters: tracer before meter
	if err := providers.TracerProvider.Shutdown(shutdownCtx); err != nil {
		logger.Error("tracer provider shutdown error", "error", err)
	}
	if err := providers.MeterProvider.Shutdown(shutdownCtx); err != nil {
		logger.Error("meter provider shutdown error", "error", err)
	}

	logger.Info("shutdown complete")
}
```

**Shutdown order:** HTTP server first (stops accepting new requests) → OTel providers (flushes buffered telemetry from completed requests). Reversing the order risks flushing incomplete spans.

**Timeout budget:** `ShutdownTimeout` (typically 25–28s when K8s terminationGracePeriodSeconds is 30s) must cover both HTTP drain and OTel flush. The OTel batcher flushes within the timeout or drops remaining data.
