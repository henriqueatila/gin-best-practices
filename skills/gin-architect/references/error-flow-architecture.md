# Error Flow Architecture Reference

How errors flow through clean architecture layers in Go Gin APIs. Covers domain error types, wrapping conventions, layer-to-layer mapping, and the complete error chain from database to HTTP response.

**Companion to:** `gin-api` skill → `error-handling.md` (HTTP handler side). This file covers the architectural view — layer boundaries, wrapping conventions, and the full chain.

## Table of Contents

1. [Error Flow Diagram](#1-error-flow-diagram)
2. [Domain Errors — Innermost Layer](#2-domain-errors--innermost-layer)
3. [Repository Layer — Translate DB Errors](#3-repository-layer--translate-db-errors)
4. [Service Layer — Wrap with Context](#4-service-layer--wrap-with-context)
5. [Handler Layer — Map to HTTP](#5-handler-layer--map-to-http)
6. [Error Wrapping Convention](#6-error-wrapping-convention)
7. [Sentinel Errors vs Custom Types](#7-sentinel-errors-vs-custom-types)
8. [errors.Is and errors.As Usage](#8-errorsis-and-errorsas-usage)
9. [Complete Error Chain Example](#9-complete-error-chain-example)
10. [Cross-Skill References](#10-cross-skill-references)

---

## 1. Error Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Error Flow (outermost → DB)                   │
│                                                                       │
│  HTTP Request                                                         │
│       │                                                               │
│       ▼                                                               │
│  [Handler]  ─── errors.Is / errors.As ──► maps to HTTP status+body   │
│       │                                                               │
│       │  returns domain.ErrNotFound (wrapped)                        │
│       ▼                                                               │
│  [Service]  ─── fmt.Errorf("Svc.Method: %w", err) ──────────────┐   │
│       │                                                           │   │
│       │  returns domain.ErrNotFound (wrapped)                    │   │
│       ▼                                                           │   │
│  [Repository] ─── translates sql.ErrNoRows → domain.ErrNotFound  │   │
│       │           fmt.Errorf("Repo.Method: %w", domainErr)       │   │
│       ▼                                                           │   │
│  [Database]   ─── sql.ErrNoRows, *pq.Error, etc.                 │   │
│                                                                   │   │
│  chain: sql.ErrNoRows → domain.ErrNotFound → "Svc: %w" → 404 ───┘   │
└──────────────────────────────────────────────────────────────────────┘
```

**Key rule:** errors flow inward → outward, wrapping at each boundary. Only the handler maps errors to HTTP. Layers below it NEVER call `c.JSON` or know about HTTP status codes.

---

## 2. Domain Errors — Innermost Layer

The domain layer owns error definitions. No external dependencies — stdlib only.

```go
// internal/user/domain/errors.go
package domain

import "errors"

// Sentinel errors — checked with errors.Is
var (
    ErrNotFound      = errors.New("not found")
    ErrAlreadyExists = errors.New("already exists")
    ErrInvalidInput  = errors.New("invalid input")
    ErrForbidden     = errors.New("forbidden")
    ErrUnauthorized  = errors.New("unauthorized")
)

// ValidationError carries field-level detail — extracted with errors.As
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return e.Field + ": " + e.Message
}

// MultiValidationError aggregates field errors
type MultiValidationError struct {
    Errors []*ValidationError
}

func (e *MultiValidationError) Error() string {
    return "validation failed: " + strconv.Itoa(len(e.Errors)) + " error(s)"
}
```

**Rules for domain errors:**
- Defined once, in the domain package
- Pure stdlib — zero third-party imports
- Sentinel errors for simple presence checks
- Typed errors for structured data (field names, codes)
- No HTTP awareness — `http.Status*` constants never appear here

---

## 3. Repository Layer — Translate DB Errors

The repository layer translates infrastructure errors (DB driver errors) into domain errors. It is the only layer that imports `database/sql`, `github.com/lib/pq`, or ORM packages.

```go
// internal/user/repository/postgres.go
package repository

import (
    "context"
    "database/sql"
    "errors"
    "fmt"

    "github.com/lib/pq"
    "yourapp/internal/user/domain"
)

func (r *postgresRepo) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    var u domain.User
    err := r.db.QueryRowContext(ctx,
        `SELECT id, email, name FROM users WHERE id = $1`, id,
    ).Scan(&u.ID, &u.Email, &u.Name)

    if err != nil {
        // Translate infrastructure error → domain error
        if errors.Is(err, sql.ErrNoRows) {
            return nil, fmt.Errorf("postgresRepo.GetByID(%d): %w", id, domain.ErrNotFound)
        }
        return nil, fmt.Errorf("postgresRepo.GetByID(%d): %w", id, err)
    }
    return &u, nil
}

func (r *postgresRepo) Create(ctx context.Context, u *domain.User) error {
    _, err := r.db.ExecContext(ctx,
        `INSERT INTO users (email, name) VALUES ($1, $2)`, u.Email, u.Name,
    )
    if err != nil {
        // Translate unique constraint violation → domain error
        var pqErr *pq.Error
        if errors.As(err, &pqErr) && pqErr.Code == "23505" {
            return fmt.Errorf("postgresRepo.Create: %w", domain.ErrAlreadyExists)
        }
        return fmt.Errorf("postgresRepo.Create: %w", err)
    }
    return nil
}
```

**Repository rules:**
- ALWAYS translate known DB errors to domain errors
- ALWAYS wrap with `fmt.Errorf("Repo.Method: %w", err)` — preserves chain
- NEVER return raw `sql.ErrNoRows` or `*pq.Error` to callers above
- NEVER log — logging is the handler's responsibility

---

## 4. Service Layer — Wrap with Context

Services implement business logic. They receive domain errors from repositories, add context, and propagate upward. They NEVER handle HTTP.

```go
// internal/user/service/user_service.go
package service

import (
    "context"
    "fmt"

    "yourapp/internal/user/domain"
)

type UserService struct {
    repo domain.UserRepository
}

func (s *UserService) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        // Add operation context, preserve error chain with %w
        return nil, fmt.Errorf("UserService.GetByID(%d): %w", id, err)
    }
    return user, nil
}

func (s *UserService) Register(ctx context.Context, req domain.RegisterRequest) (*domain.User, error) {
    // Business rule validation — return domain error
    if req.Password != req.PasswordConfirm {
        return nil, fmt.Errorf("UserService.Register: %w",
            &domain.ValidationError{Field: "password_confirm", Message: "passwords do not match"},
        )
    }

    user := &domain.User{Email: req.Email, Name: req.Name}
    if err := s.repo.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("UserService.Register: %w", err)
    }
    return user, nil
}
```

**Service rules:**
- ALWAYS use `%w` — never `%v` — so errors.Is/As work through the chain
- Add meaningful context: operation name, key parameter values
- NEVER call `c.JSON`, reference `net/http` status codes, or log
- Domain error types pass through transparently via wrapping

---

## 5. Handler Layer — Map to HTTP

Handlers map domain errors to HTTP responses. This is the ONLY layer that knows about HTTP status codes.

```go
// internal/user/handler/user_handler.go
package handler

import (
    "errors"
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "yourapp/internal/user/domain"
    "yourapp/internal/user/service"
)

type UserHandler struct {
    svc *service.UserService
}

func (h *UserHandler) GetUser(c *gin.Context) {
    id, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
        return
    }

    user, err := h.svc.GetByID(c.Request.Context(), id)
    if err != nil {
        status, msg := mapDomainError(err)

        // Handler is the only layer that logs
        if status == http.StatusInternalServerError {
            slog.ErrorContext(c.Request.Context(), "unexpected error",
                "op", "GetUser",
                "id", id,
                "err", err,
            )
        }

        c.JSON(status, gin.H{"error": msg})
        return
    }

    c.JSON(http.StatusOK, user)
}

// mapDomainError converts domain errors to HTTP status + message pairs.
// Single responsibility: this function is the entire mapping table.
func mapDomainError(err error) (int, string) {
    // Check typed errors first (errors.As) before sentinel errors (errors.Is)
    var valErr *domain.ValidationError
    if errors.As(err, &valErr) {
        return http.StatusBadRequest, valErr.Error()
    }

    var multiErr *domain.MultiValidationError
    if errors.As(err, &multiErr) {
        return http.StatusBadRequest, multiErr.Error()
    }

    switch {
    case errors.Is(err, domain.ErrNotFound):
        return http.StatusNotFound, "resource not found"
    case errors.Is(err, domain.ErrAlreadyExists):
        return http.StatusConflict, "resource already exists"
    case errors.Is(err, domain.ErrInvalidInput):
        return http.StatusBadRequest, "invalid input"
    case errors.Is(err, domain.ErrForbidden):
        return http.StatusForbidden, "access denied"
    case errors.Is(err, domain.ErrUnauthorized):
        return http.StatusUnauthorized, "unauthorized"
    default:
        return http.StatusInternalServerError, "internal error"
    }
}
```

**Handler rules:**
- NEVER wrap errors — only map them
- Log only at handler level (not service, not repository)
- Log only unexpected errors (5xx); 4xx errors are normal flow
- `mapDomainError` is a pure function — easy to test independently

---

## 6. Error Wrapping Convention

| Layer      | Convention                                     | Purpose                            |
|------------|------------------------------------------------|------------------------------------|
| Domain     | `errors.New("description")`                    | Define sentinel/root errors        |
| Repository | `fmt.Errorf("Repo.Method(args): %w", err)`     | Translate + add call site          |
| Service    | `fmt.Errorf("Service.Method(args): %w", err)`  | Add operation context              |
| Handler    | No wrapping — call `mapDomainError(err)`        | Translate to HTTP                  |

**Always use `%w`, never `%v`.**

`%v` stringifies the error — `errors.Is` and `errors.As` cannot unwrap it. `%w` wraps it — the chain stays traversable.

```go
// CORRECT — chain preserved
return fmt.Errorf("UserService.GetByID(%d): %w", id, err)

// WRONG — chain broken, errors.Is will not find domain.ErrNotFound
return fmt.Errorf("UserService.GetByID(%d): %v", id, err)
```

---

## 7. Sentinel Errors vs Custom Types

**Use sentinel errors when** the caller only needs to know *what happened*:

```go
var ErrNotFound = errors.New("not found")

// Caller
if errors.Is(err, domain.ErrNotFound) {
    // handle
}
```

**Use custom error types when** the caller needs structured data:

```go
type ValidationError struct {
    Field   string
    Message string
}
func (e *ValidationError) Error() string { return e.Field + ": " + e.Message }

// Caller
var ve *domain.ValidationError
if errors.As(err, &ve) {
    // access ve.Field and ve.Message
}
```

**Decision rule:** start with sentinels. Add a custom type only when you need to attach data that callers must read programmatically.

---

## 8. errors.Is and errors.As Usage

Both functions traverse the entire wrapping chain automatically.

```go
// errors.Is — checks identity/equality through the chain
// Works for sentinel errors and any error that implements Is(target error) bool
errors.Is(err, domain.ErrNotFound) // true even if err is 3 levels deep

// errors.As — extracts the first matching type from the chain
// Assigns to the target pointer if found
var valErr *domain.ValidationError
if errors.As(err, &valErr) {
    slog.Warn("validation failed", "field", valErr.Field, "msg", valErr.Message)
}

// Checking order matters — check specific types before broad sentinels
var ve *domain.ValidationError
switch {
case errors.As(err, &ve):      // check typed first
    return http.StatusBadRequest, ve.Message
case errors.Is(err, domain.ErrNotFound):
    return http.StatusNotFound, "not found"
}
```

**Pitfall:** never unwrap manually with `.(type)` assertions — they do not traverse wrapped chains. Always use `errors.Is` / `errors.As`.

---

## 9. Complete Error Chain Example

Tracing a "get user by ID" request where the record does not exist.

**Step 1 — Database returns no rows:**
```go
// database/sql internal
err = sql.ErrNoRows
```

**Step 2 — Repository translates and wraps:**
```go
// postgresRepo.GetByID
if errors.Is(err, sql.ErrNoRows) {
    return nil, fmt.Errorf("postgresRepo.GetByID(42): %w", domain.ErrNotFound)
}
// err chain: "postgresRepo.GetByID(42): not found"
//            wraps → domain.ErrNotFound
```

**Step 3 — Service adds context and wraps:**
```go
// UserService.GetByID
return nil, fmt.Errorf("UserService.GetByID(42): %w", err)
// err chain: "UserService.GetByID(42): postgresRepo.GetByID(42): not found"
//            wraps → domain.ErrNotFound  (still reachable via errors.Is)
```

**Step 4 — Handler maps to HTTP:**
```go
// UserHandler.GetUser
status, msg := mapDomainError(err)
// errors.Is(err, domain.ErrNotFound) → true  (chain traversed automatically)
// status = 404, msg = "resource not found"

c.JSON(http.StatusNotFound, gin.H{"error": "resource not found"})
```

**Step 5 — Handler logs unexpected errors only:**
```go
// 404 is expected — no log
// If err were an unmapped error:
slog.ErrorContext(ctx, "unexpected error", "op", "GetUser", "id", 42, "err", err)
```

**Full error string visible in logs (5xx path):**
```
UserService.GetByID(42): postgresRepo.GetByID(42): not found
```
Readable stack of context without a single `%v` that would break the chain.

---

## 10. Cross-Skill References

| Skill / File | What It Covers |
|---|---|
| `gin-api` → `error-handling.md` | AppError struct, panic recovery middleware, validation error binding (HTTP handler side) |
| `gin-architect` → `clean-architecture.md` | Layer boundaries, dependency rule, interface definitions |
| `gin-architect` → `cross-cutting-concerns.md` | Logging strategy, middleware placement, observability |
| `gin-database` → `repository-pattern.md` | Repository interface definition, sqlx/GORM implementations |
| `gin-testing` | Testing `mapDomainError`, error path coverage in service unit tests |

**Summary of responsibilities:**

```
domain/errors.go       → define error types (no deps)
repository/*.go        → translate DB errors → domain errors, wrap with context
service/*.go           → wrap errors with operation context, propagate
handler/error_map.go   → single mapDomainError function, log 5xx, call c.JSON
```
