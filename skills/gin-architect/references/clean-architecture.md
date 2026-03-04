# Clean Architecture Reference

Practical guide to applying Clean Architecture principles in Go Gin APIs. Maps Uncle Bob's theory to Go-idiomatic patterns — interfaces as ports, packages as boundaries, the compiler as your architecture enforcer.

**Default stance:** Start with flat package layout. Adopt clean architecture layering when the codebase outgrows it — not before. See [system-design.md](system-design.md) for project structure by scale.

## Table of Contents

1. [Layer Mapping: Uncle Bob → Go](#1-layer-mapping-uncle-bob--go)
2. [The Dependency Rule in Go](#2-the-dependency-rule-in-go)
3. [Ports & Adapters (Hexagonal)](#3-ports--adapters-hexagonal)
4. [Wiring It All Together (DI)](#4-wiring-it-all-together-di)
5. [Complete Feature Module Example](#5-complete-feature-module-example)
6. [Common Mistakes](#6-common-mistakes)
7. [When to Use What](#7-when-to-use-what)

---

## 1. Layer Mapping: Uncle Bob → Go

Uncle Bob's 4 concentric rings translate to Go packages:

```
┌─────────────────────────────────────────────────────┐
│  cmd/api/main.go          ← Wiring (imports all)    │
│  ┌─────────────────────────────────────────────┐    │
│  │  handler/              ← Driving Adapters    │    │
│  │  (Gin HTTP handlers)   (Framework layer)     │    │
│  │  ┌─────────────────────────────────────┐    │    │
│  │  │  usecase/ or service/               │    │    │
│  │  │  (Business logic + port interfaces) │    │    │
│  │  │  ┌─────────────────────────────┐    │    │    │
│  │  │  │  domain/                    │    │    │    │
│  │  │  │  (Entities, value objects)  │    │    │    │
│  │  │  │  stdlib only, zero deps     │    │    │    │
│  │  │  └─────────────────────────────┘    │    │    │
│  │  └─────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────┘    │
│  repository/              ← Driven Adapters          │
│  (sqlx/GORM implementations of port interfaces)     │
└─────────────────────────────────────────────────────┘
```

| Uncle Bob Layer | Go Package | Imports | Contains |
|---|---|---|---|
| Entities | `domain/` | stdlib only | Structs, value objects, domain errors, validation rules |
| Use Cases | `usecase/` or `service/` | `domain/` only | Business logic, port interfaces (Repository, Notifier, etc.) |
| Interface Adapters | `handler/` (driving), `repository/` (driven) | `usecase/`, `domain/` | Gin handlers, sqlx repos, external API clients |
| Frameworks & Drivers | `cmd/api/main.go` | everything | Wiring, config, server startup |

**Go-specific twist:** Interfaces are defined in the **consuming** package (use case), not the implementing package (repository). This is idiomatic Go and structurally enforces the dependency rule.

---

## 2. The Dependency Rule in Go

> Inner layers NEVER import outer layers. Dependencies point inward.

Go's compiler enforces this. If `domain/` tries to import `repository/`, you get a circular import error. This is a built-in architecture guardrail — use it.

```
domain/    → imports: nothing (stdlib only)
usecase/   → imports: domain/
handler/   → imports: usecase/, domain/
repository/→ imports: usecase/ (for interface), domain/ (for entities)
cmd/       → imports: everything (wiring only)
```

**Compile-time interface check** — always add this in the adapter package:

```go
// repository/postgres/user_repository.go
var _ usecase.UserRepository = (*UserRepository)(nil)
```

This fails at compile time if `UserRepository` doesn't satisfy the interface. Zero runtime cost.

---

## 3. Ports & Adapters (Hexagonal)

A **port** is a Go interface. An **adapter** is an implementation.

- **Driving adapter** = something that calls INTO your app (Gin handler, CLI, gRPC server)
- **Driven adapter** = something your app calls OUT to (database, external API, message queue)

### Port: defined in usecase package

```go
// internal/user/usecase/ports.go
package usecase

import (
    "context"

    "myapp/internal/user/domain"
)

// UserRepository is the driven port for user persistence.
type UserRepository interface {
    GetByID(ctx context.Context, id int64) (*domain.User, error)
    Create(ctx context.Context, user *domain.User) error
    GetByEmail(ctx context.Context, email string) (*domain.User, error)
}
```

### Driven adapter: sqlx implementation

```go
// internal/user/repository/postgres/user_repository.go
package postgres

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"

    "myapp/internal/user/domain"
    "myapp/internal/user/usecase"
)

var _ usecase.UserRepository = (*UserRepository)(nil)

type UserRepository struct {
    db *sqlx.DB
}

func NewUserRepository(db *sqlx.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    var u domain.User
    err := r.db.GetContext(ctx, &u, `SELECT id, email, name, created_at FROM users WHERE id = $1`, id)
    if err != nil {
        return nil, fmt.Errorf("UserRepository.GetByID: %w", err)
    }
    return &u, nil
}

func (r *UserRepository) Create(ctx context.Context, user *domain.User) error {
    _, err := r.db.NamedExecContext(ctx, `
        INSERT INTO users (email, name, password_hash)
        VALUES (:email, :name, :password_hash)`, user)
    if err != nil {
        return fmt.Errorf("UserRepository.Create: %w", err)
    }
    return nil
}

func (r *UserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    var u domain.User
    err := r.db.GetContext(ctx, &u, `SELECT id, email, name, password_hash, created_at FROM users WHERE email = $1`, email)
    if err != nil {
        return nil, fmt.Errorf("UserRepository.GetByEmail: %w", err)
    }
    return &u, nil
}
```

### Driving adapter: Gin handler

```go
// internal/user/handler/user_handler.go
package handler

import (
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"

    "myapp/internal/user/usecase"
)

type UserHandler struct {
    svc *usecase.UserService
}

func NewUserHandler(svc *usecase.UserService) *UserHandler {
    return &UserHandler{svc: svc}
}

func (h *UserHandler) GetByID(c *gin.Context) {
    id, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid user id"})
        return
    }
    user, err := h.svc.GetByID(c.Request.Context(), id)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
        return
    }
    c.JSON(http.StatusOK, user)
}
```

---

## 4. Wiring It All Together (DI)

**Manual DI in `main.go`** is the Go-idiomatic choice. Wire/fx only justifies itself at 20+ services.

```go
// cmd/api/main.go
package main

import (
    "log/slog"
    "os"

    "github.com/gin-gonic/gin"
    "github.com/jmoiron/sqlx"

    userHandler    "myapp/internal/user/handler"
    userPostgres   "myapp/internal/user/repository/postgres"
    userUsecase    "myapp/internal/user/usecase"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    db, err := sqlx.Connect("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        logger.Error("database connection failed", "error", err)
        os.Exit(1)
    }

    // Wire dependencies: repo → service → handler
    userRepo := userPostgres.NewUserRepository(db)
    userSvc  := userUsecase.NewUserService(userRepo, logger)
    userH    := userHandler.NewUserHandler(userSvc)

    r := gin.New()
    r.Use(gin.Recovery())

    api := r.Group("/api/v1")
    api.GET("/users/:id", userH.GetByID)

    r.Run(":8080")
}
```

**The wiring rule:** `main.go` is the only file that knows about ALL packages. Everything else only knows its inner layers.

---

## 5. Complete Feature Module Example

For medium+ projects using feature modules (recommended default):

```
internal/
├── user/
│   ├── domain/
│   │   ├── user.go           # Entity: User struct, validation
│   │   └── errors.go         # Domain errors: ErrUserNotFound, ErrDuplicateEmail
│   ├── usecase/
│   │   ├── ports.go          # Interface: UserRepository
│   │   └── user_service.go   # Business logic
│   ├── handler/
│   │   └── user_handler.go   # Gin HTTP handlers
│   └── repository/
│       └── postgres/
│           └── user_repository.go  # sqlx implementation
├── order/
│   ├── domain/
│   ├── usecase/
│   ├── handler/
│   └── repository/
└── platform/                  # Shared infrastructure (not a domain)
    ├── database/              # DB connection factory
    ├── middleware/             # Auth, logging, CORS
    └── config/                # Environment config
```

### Domain layer

```go
// internal/user/domain/user.go
package domain

import "time"

type User struct {
    ID           int64     `db:"id" json:"id"`
    Email        string    `db:"email" json:"email"`
    Name         string    `db:"name" json:"name"`
    PasswordHash string    `db:"password_hash" json:"-"`
    CreatedAt    time.Time `db:"created_at" json:"created_at"`
}

// internal/user/domain/errors.go
package domain

import "errors"

var (
    ErrUserNotFound   = errors.New("user not found")
    ErrDuplicateEmail = errors.New("email already registered")
)
```

### Use case layer

```go
// internal/user/usecase/user_service.go
package usecase

import (
    "context"
    "fmt"
    "log/slog"

    "myapp/internal/user/domain"
)

type UserService struct {
    repo   UserRepository
    logger *slog.Logger
}

func NewUserService(repo UserRepository, logger *slog.Logger) *UserService {
    return &UserService{
        repo:   repo,
        logger: logger.With("component", "user-service"),
    }
}

func (s *UserService) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("UserService.GetByID: %w", err)
    }
    return user, nil
}
```

---

## 6. Common Mistakes

| Mistake | Why it's wrong | Fix |
|---|---|---|
| Interface next to implementation | Couples consumer to provider | Interface in `usecase/`, implementation in `repository/` |
| Domain type as JSON response | Leaks DB fields, breaks API contract on schema changes | Use response DTOs in `handler/` |
| Business logic in handlers | Untestable without HTTP, violates SRP | Handlers: bind → call service → respond. Nothing else. |
| Interface for everything | Over-abstraction, Go interfaces should be small and discovered | Only where you need to swap implementations (repos yes, simple helpers no) |
| Clean arch for a 5-endpoint MVP | Over-engineering, YAGNI | Start flat (`handler/`, `service/`, `repository/`). Refactor when it hurts. |
| Shared DB between services | Defeats independent deployment, hidden coupling | Each service owns its data. Communicate via API. |
| `domain/` importing `gin` or `sqlx` | Violates dependency rule | Domain is stdlib-only. Gin stays in `handler/`, sqlx stays in `repository/`. |

---

## 7. When to Use What

| Project Size | Recommended Structure | Clean Arch Level |
|---|---|---|
| MVP, < 10 endpoints, 1-2 devs | Flat: `handler/`, `service/`, `repository/`, `domain/` | Implicit (just layers, no formal ports) |
| Growing, 10-50 endpoints, 3-8 devs | Feature modules: `internal/user/`, `internal/order/` | Per-module: ports in `usecase/`, adapters separate |
| Complex domain, 50+ endpoints, 8+ devs | Full clean arch with bounded contexts | Strict: dependency rule enforced, DTOs at boundaries |

**Rule of thumb:** Start flat. When you feel pain (circular deps, fat handlers, tests requiring real DB), introduce the next level of structure. The moment you add the second adapter for an interface (e.g., in-memory repo for tests + postgres for prod), clean architecture has paid for itself.

### Gate — you need formal clean architecture when

- [ ] You have 2+ delivery mechanisms (HTTP + gRPC + CLI) calling the same business logic
- [ ] You need to swap infrastructure (e.g., migrate from MySQL to PostgreSQL) without touching business logic
- [ ] Multiple teams work on the same service and need clear ownership boundaries
- [ ] Your test setup requires complex mocking because business logic is tangled with HTTP/DB

If none of these apply, the flat layout with implicit layering (handler → service → repository) is clean architecture in spirit — just without the formal ceremony.

---

## Cross-Skill References

- For project structure at different scales: see **[system-design.md](system-design.md)** (package layouts, bounded contexts)
- For complexity budget (is this overkill?): see **[complexity-assessment.md](complexity-assessment.md)**
- For repository and ORM patterns: see the **gin-database** skill
- For handler and middleware implementation: see the **gin-api** skill
- For testing layers in isolation: see the **gin-testing** skill
