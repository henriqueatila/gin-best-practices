# Golden main.go Template Reference

The complete, production-ready `cmd/api/main.go` that wires all layers together. Every Gin project starts here — covers config loading, database connection, dependency injection, middleware stack, graceful shutdown, and signal handling.

## Table of Contents

1. [Complete main.go (Small Project)](#1-complete-maingo-small-project)
2. [Complete main.go (Medium Project with Feature Modules)](#2-complete-maingo-medium-project-with-feature-modules)
3. [Startup Sequence Explained](#3-startup-sequence-explained)
4. [Graceful Shutdown Pattern](#4-graceful-shutdown-pattern)
5. [Cross-Skill References](#5-cross-skill-references)

---

## 1. Complete main.go (Small Project)

Single-domain app (e.g. a users API). All wiring in one place, no feature module splitting.

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/jmoiron/sqlx"
	_ "github.com/lib/pq"

	"myapp/internal/config"
	"myapp/internal/handler"
	"myapp/internal/repository"
	"myapp/internal/service"
	"myapp/pkg/middleware"
)

func main() {
	// 1. Logger — initialized first so everything else can log
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))

	// 2. Config — loaded second so everything else reads from it
	cfg, err := config.Load()
	if err != nil {
		logger.Error("failed to load config", "error", err)
		os.Exit(1)
	}

	// 3. Database — third, repositories depend on it
	db, err := sqlx.Connect("postgres", cfg.DatabaseURL)
	if err != nil {
		logger.Error("failed to connect to database", "error", err)
		os.Exit(1)
	}
	db.SetMaxOpenConns(cfg.DBMaxOpenConns)
	db.SetMaxIdleConns(cfg.DBMaxIdleConns)
	db.SetConnMaxLifetime(cfg.DBConnMaxLifetime)
	defer db.Close()

	// 4. Wire dependencies — manual DI: repo → service → handler
	userRepo := repository.NewUserRepository(db)
	userSvc := service.NewUserService(userRepo, logger)
	userHandler := handler.NewUserHandler(userSvc, logger)
	healthHandler := handler.NewHealthHandler(db, logger)

	// 5. Router — gin.New() never gin.Default()
	if cfg.GinMode == "release" {
		gin.SetMode(gin.ReleaseMode)
	}
	r := gin.New()
	r.Use(gin.Recovery())
	r.Use(middleware.RequestID())
	r.Use(middleware.Logger(logger))
	r.Use(middleware.CORS(cfg.AllowedOrigins))

	// 6. Routes
	r.GET("/health", healthHandler.Check)

	api := r.Group("/api/v1")
	api.GET("/users/:id", userHandler.GetByID)
	api.POST("/users", userHandler.Create)
	api.PUT("/users/:id", userHandler.Update)
	api.DELETE("/users/:id", userHandler.Delete)

	// 7. HTTP server with explicit timeouts
	srv := &http.Server{
		Addr:         ":" + cfg.Port,
		Handler:      r,
		ReadTimeout:  cfg.ReadTimeout,
		WriteTimeout: cfg.WriteTimeout,
		IdleTimeout:  cfg.IdleTimeout,
	}

	// 8. Start server in goroutine — main goroutine blocks on signal
	go func() {
		logger.Info("server starting", "port", cfg.Port, "mode", cfg.GinMode)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			logger.Error("server error", "error", err)
			os.Exit(1)
		}
	}()

	// 9. Block until OS signal received
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	logger.Info("shutdown signal received")

	// 10. Graceful shutdown — drain in-flight requests within timeout
	ctx, cancel := context.WithTimeout(context.Background(), cfg.ShutdownTimeout)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		logger.Error("server forced shutdown", "error", err)
		os.Exit(1)
	}
	logger.Info("server stopped")
}
```

**Config struct expected** (`internal/config/config.go`):

```go
type Config struct {
	Port              string
	GinMode           string
	DatabaseURL       string
	DBMaxOpenConns    int
	DBMaxIdleConns    int
	DBConnMaxLifetime time.Duration
	ReadTimeout       time.Duration
	WriteTimeout      time.Duration
	IdleTimeout       time.Duration
	ShutdownTimeout   time.Duration
	AllowedOrigins    []string
}

func Load() (*Config, error) {
	return &Config{
		Port:              getEnv("PORT", "8080"),
		GinMode:           getEnv("GIN_MODE", "debug"),
		DatabaseURL:       mustEnv("DATABASE_URL"),
		DBMaxOpenConns:    getEnvInt("DB_MAX_OPEN_CONNS", 25),
		DBMaxIdleConns:    getEnvInt("DB_MAX_IDLE_CONNS", 5),
		DBConnMaxLifetime: getEnvDuration("DB_CONN_MAX_LIFETIME", 5*time.Minute),
		ReadTimeout:       getEnvDuration("READ_TIMEOUT", 10*time.Second),
		WriteTimeout:      getEnvDuration("WRITE_TIMEOUT", 10*time.Second),
		IdleTimeout:       getEnvDuration("IDLE_TIMEOUT", 60*time.Second),
		ShutdownTimeout:   getEnvDuration("SHUTDOWN_TIMEOUT", 30*time.Second),
	}, nil
}
```

---

## 2. Complete main.go (Medium Project with Feature Modules)

Multiple domains (users, orders, products). Each feature module owns its handler and route registration.

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"

	"github.com/gin-gonic/gin"
	"github.com/jmoiron/sqlx"
	_ "github.com/lib/pq"

	"myapp/internal/config"
	"myapp/internal/features/order/orderhandler"
	"myapp/internal/features/order/orderpostgres"
	"myapp/internal/features/order/orderusecase"
	"myapp/internal/features/user/userhandler"
	"myapp/internal/features/user/userpostgres"
	"myapp/internal/features/user/userusecase"
	"myapp/pkg/middleware"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))

	cfg, err := config.Load()
	if err != nil {
		logger.Error("failed to load config", "error", err)
		os.Exit(1)
	}

	db, err := sqlx.Connect("postgres", cfg.DatabaseURL)
	if err != nil {
		logger.Error("failed to connect to database", "error", err)
		os.Exit(1)
	}
	db.SetMaxOpenConns(cfg.DBMaxOpenConns)
	db.SetMaxIdleConns(cfg.DBMaxIdleConns)
	db.SetConnMaxLifetime(cfg.DBConnMaxLifetime)
	defer db.Close()

	// Wire each feature module independently — no cross-module repo sharing
	userRepo := userpostgres.NewRepository(db)
	userSvc := userusecase.NewService(userRepo, logger)
	userH := userhandler.NewHandler(userSvc, logger)

	orderRepo := orderpostgres.NewRepository(db)
	orderSvc := orderusecase.NewService(orderRepo, userSvc, logger)
	orderH := orderhandler.NewHandler(orderSvc, logger)

	// Router
	if cfg.GinMode == "release" {
		gin.SetMode(gin.ReleaseMode)
	}
	r := gin.New()
	r.Use(gin.Recovery())
	r.Use(middleware.RequestID())
	r.Use(middleware.Logger(logger))

	// Health (no auth)
	r.GET("/health", func(c *gin.Context) {
		if err := db.PingContext(c.Request.Context()); err != nil {
			c.JSON(http.StatusServiceUnavailable, gin.H{"status": "unhealthy"})
			return
		}
		c.JSON(http.StatusOK, gin.H{"status": "ok"})
	})

	// Authenticated API group
	authMiddleware := middleware.JWT(cfg.JWTSecret)
	api := r.Group("/api/v1")
	api.Use(authMiddleware)

	// Each handler package registers its own routes
	userhandler.RegisterRoutes(api, userH)
	orderhandler.RegisterRoutes(api, orderH)

	srv := &http.Server{
		Addr:         ":" + cfg.Port,
		Handler:      r,
		ReadTimeout:  cfg.ReadTimeout,
		WriteTimeout: cfg.WriteTimeout,
		IdleTimeout:  cfg.IdleTimeout,
	}

	go func() {
		logger.Info("server starting", "port", cfg.Port)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			logger.Error("server error", "error", err)
			os.Exit(1)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	logger.Info("shutdown signal received")

	ctx, cancel := context.WithTimeout(context.Background(), cfg.ShutdownTimeout)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		logger.Error("server forced shutdown", "error", err)
		os.Exit(1)
	}
	logger.Info("server stopped")
}
```

**RegisterRoutes pattern** — each handler package owns its route registration:

```go
// internal/features/user/userhandler/routes.go
package userhandler

import "github.com/gin-gonic/gin"

func RegisterRoutes(rg *gin.RouterGroup, h *Handler) {
	users := rg.Group("/users")
	users.GET("/:id", h.GetByID)
	users.POST("", h.Create)
	users.PUT("/:id", h.Update)
	users.DELETE("/:id", h.Delete)
}

// internal/features/order/orderhandler/routes.go
package orderhandler

import (
	"github.com/gin-gonic/gin"
	"myapp/pkg/middleware"
)

func RegisterRoutes(rg *gin.RouterGroup, h *Handler, authMiddleware gin.HandlerFunc) {
	orders := rg.Group("/orders")
	orders.GET("/:id", h.GetByID)
	orders.POST("", h.Create)
	// Admin-only sub-group
	admin := orders.Group("", middleware.RequireRole("admin"))
	admin.DELETE("/:id", h.Delete)
}
```

---

## 3. Startup Sequence Explained

Order is strict. Each step depends on what came before.

```
1. Logger
   └── reason: every subsequent step needs to log errors

2. Config
   └── reason: all other components read from config
   └── fail fast: os.Exit(1) if missing required env vars

3. Database
   └── reason: repositories depend on *sqlx.DB
   └── configure pool immediately after connect
   └── fail fast: os.Exit(1) on connection failure

4. Repositories
   └── reason: services depend on repository interfaces
   └── inject *sqlx.DB

5. Services
   └── reason: handlers depend on service interfaces
   └── inject repositories + logger

6. Handlers
   └── reason: routes depend on handler methods
   └── inject services + logger

7. Router + Middleware
   └── gin.New() — never gin.Default()
   └── global middleware before routes: Recovery → RequestID → Logger → CORS

8. Routes
   └── register health endpoint (no auth)
   └── register authenticated groups
   └── call RegisterRoutes per feature module

9. http.Server
   └── set explicit timeouts — never use zero values
   └── start in goroutine

10. Signal handling
    └── block main goroutine on quit channel
    └── syscall.SIGINT + syscall.SIGTERM
    └── graceful shutdown with context timeout
```

**Why `gin.New()` not `gin.Default()`:**

`gin.Default()` adds a logger that writes to stdout in a non-structured format, conflicting with `slog`. It also adds `Recovery()` silently. Use `gin.New()` and add middleware explicitly so the middleware stack is auditable.

---

## 4. Graceful Shutdown Pattern

Shutdown ordering matters: stop accepting traffic first, drain in-flight, then close resources.

```
Signal received (SIGINT / SIGTERM)
        │
        ▼
srv.Shutdown(ctx)          ← stops accepting new connections
        │                    drains in-flight requests
        │                    blocks until done or ctx expires
        ▼
db.Close()                 ← called via defer after Shutdown returns
        │
        ▼
logger.Info("stopped")     ← final log line, then process exits 0
```

**Full shutdown with resource cleanup:**

```go
// Signal wait
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit
logger.Info("shutdown signal received")

// Stop accepting connections, drain in-flight
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
	// ctx expired before drain completed — forced close
	logger.Error("shutdown timeout, forcing close", "error", err)
	_ = srv.Close()
}

// DB closed by deferred db.Close() after main() returns
logger.Info("server stopped")
```

**Rules:**
- `ShutdownTimeout` must be greater than your longest expected request duration
- Never call `srv.Close()` before `srv.Shutdown()` — `Close()` drops connections immediately
- `db.Close()` via `defer` runs after `Shutdown()` returns, so in-flight DB queries complete first
- Log the final "stopped" line — confirms clean exit in production log pipelines

**Context propagation during shutdown:**

Handlers must propagate `c.Request.Context()` to all downstream calls. When `Shutdown()` is called, the server cancels request contexts after the drain period, allowing graceful termination of long-running DB queries.

```go
// handler — correct
func (h *UserHandler) GetByID(c *gin.Context) {
	user, err := h.svc.GetByID(c.Request.Context(), c.Param("id"))
	// ...
}

// service — correct
func (s *UserService) GetByID(ctx context.Context, id string) (*User, error) {
	return s.repo.FindByID(ctx, id)
}

// repository — correct
func (r *UserRepository) FindByID(ctx context.Context, id string) (*User, error) {
	var user User
	err := r.db.GetContext(ctx, &user, "SELECT * FROM users WHERE id = $1", id)
	return &user, err
}
```

---

## 5. Cross-Skill References

| Topic | Skill |
|---|---|
| Handler patterns, `ShouldBind*`, response helpers | `gin-api` |
| Repository patterns, `sqlx.GetContext`, migrations | `gin-database` |
| JWT middleware, auth groups | `gin-auth` |
| Docker, env config, health checks for k8s | `gin-deploy` |
| Clean architecture folder structure | `references/clean-architecture.md` |
| Middleware implementation details | `references/cross-cutting-concerns.md` |

**Conventions enforced in this template:**

- `gin.New()` — never `gin.Default()`
- `log/slog` with JSON handler — structured, parseable in production
- `ShouldBind*` in handlers (never `Bind*` which writes 400 automatically)
- `c.Request.Context()` propagated to every service and repo call
- `context.WithTimeout` wraps graceful shutdown
- `syscall.SIGINT, syscall.SIGTERM` — both signals handled
- Manual DI — no DI framework; wire order is explicit and readable
- `http.Server` timeouts always set — `ReadTimeout`, `WriteTimeout`, `IdleTimeout`
- Pool configured immediately after `sqlx.Connect` — before any query runs
