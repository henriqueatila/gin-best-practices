# Error Handling Reference

This file covers the full domain error system for Gin APIs: AppError struct, sentinel errors, service error mapping, validation error formatting, panic recovery, and consistent JSON response format. Use this when implementing error responses beyond the basics shown in gin-api/SKILL.md.

## Table of Contents

1. [AppError Struct](#apperror-struct)
2. [Sentinel Errors](#sentinel-errors)
3. [handleServiceError Function](#handleserviceerror-function)
4. [Error Wrapping & Unwrapping](#error-wrapping--unwrapping)
5. [Validation Error Formatting](#validation-error-formatting)
6. [Panic Recovery Middleware](#panic-recovery-middleware)
7. [Consistent JSON Error Format](#consistent-json-error-format)

---

## AppError Struct

`AppError` carries an HTTP status code alongside the message, so the transport layer never decides status codes — the domain does.

> **Note:** This `AppError` is a simplified version for this skill's examples. For the canonical domain error pattern with `Detail` field and 5xx guard, see the **golang-gin-clean-arch** skill.

```go
// internal/domain/errors.go
package domain

import "errors"

// AppError is a domain error that maps directly to an HTTP response.
// Services return *AppError; handlers use handleServiceError to render it.
type AppError struct {
    Code    int    // HTTP status code
    Message string // User-facing message (safe to expose)
    Err     error  // Wrapped cause (for logging, not exposed to client)
}

func (e *AppError) Error() string { return e.Message }
func (e *AppError) Unwrap() error  { return e.Err }
func (e AppError) Is(target error) bool {
    switch t := target.(type) {
    case *AppError:
        return e.Code == t.Code
    case AppError:
        return e.Code == t.Code
    }
    return false
}

// New wraps an underlying cause with a domain error.
func (e *AppError) New(cause error) *AppError {
    return &AppError{Code: e.Code, Message: e.Message, Err: cause}
}
```

**Why:** Keeping HTTP status codes in the domain layer means handler code stays thin. The service returns `domain.ErrNotFound.New(err)` and the handler maps it without knowing the HTTP status.

---

## Sentinel Errors

Define one sentinel per failure mode. Services return these (wrapped with cause if needed).

```go
// internal/domain/errors.go (continued)
package domain

var (
    // ErrNotFound is returned when a requested resource does not exist.
    ErrNotFound = &AppError{Code: 404, Message: "resource not found"}

    // ErrUnauthorized is returned when authentication is missing or invalid.
    ErrUnauthorized = &AppError{Code: 401, Message: "unauthorized"}

    // ErrForbidden is returned when the user lacks permission for the action.
    ErrForbidden = &AppError{Code: 403, Message: "forbidden"}

    // ErrConflict is returned when a unique constraint would be violated.
    ErrConflict = &AppError{Code: 409, Message: "resource already exists"}

    // ErrValidation is returned when business rule validation fails (not binding).
    ErrValidation = &AppError{Code: 422, Message: "validation failed"}

    // ErrInternal is returned for unexpected errors. Do NOT expose the cause.
    ErrInternal = &AppError{Code: 500, Message: "internal server error"}
)
```

Usage in a service:

```go
// internal/service/user_service.go
func (s *userService) GetByID(ctx context.Context, id string) (*domain.User, error) {
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return user, nil
}

func (s *userService) Create(ctx context.Context, req domain.CreateUserRequest) (*domain.User, error) {
    existing, err := s.repo.GetByEmail(ctx, req.Email)
    if err != nil && !errors.Is(err, sql.ErrNoRows) {
        return nil, domain.ErrInternal.New(err)
    }
    if existing != nil {
        return nil, domain.ErrConflict.New(fmt.Errorf("email %s already registered", req.Email))
    }
    // ...
}
```

---

## handleServiceError Function

Place this in the handler package — it is the single translation point between domain errors and HTTP responses.

```go
// internal/handler/errors.go
package handler

import (
    "errors"
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
)

// handleServiceError maps a domain error to a JSON HTTP response.
// All unrecognised errors become 500 — the cause is logged, NOT sent to client.
func handleServiceError(c *gin.Context, err error, logger *slog.Logger) {
    var appErr *domain.AppError
    if errors.As(err, &appErr) {
        if appErr.Code >= 500 {
            logger.ErrorContext(c.Request.Context(), "service error",
                "error", appErr.Unwrap(),
                "path", c.FullPath(),
            )
        }
        c.JSON(appErr.Code, gin.H{"error": appErr.Message})
        return
    }

    // Unexpected error — log it, return generic message
    logger.ErrorContext(c.Request.Context(), "unexpected error",
        "error", err,
        "path", c.FullPath(),
    )
    c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
}
```

Handler usage:

> **Security note:** In production, never expose raw `err.Error()` to clients. Return generic messages and log the error server-side. See **golang-gin-clean-arch** error handling patterns.

```go
func (h *UserHandler) GetByID(c *gin.Context) {
    type uriParams struct {
        ID string `uri:"id" binding:"required"`
    }
    var p uriParams
    if err := c.ShouldBindURI(&p); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.svc.GetByID(c.Request.Context(), p.ID)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, user)
}
```

---

## Error Wrapping & Unwrapping

Use `fmt.Errorf` with `%w` to wrap errors while preserving the chain. Use `errors.As` (not type assertion) to unwrap.

```go
// Wrapping — preserves the chain
return nil, fmt.Errorf("userService.Create: %w", domain.ErrConflict.New(err))

// Unwrapping — errors.As walks the chain
var appErr *domain.AppError
if errors.As(err, &appErr) {
    // appErr is the first *AppError in the chain
}

// errors.Is for sentinel comparison (requires Unwrap() on AppError)
if errors.Is(err, domain.ErrNotFound) {
    // true if ErrNotFound appears anywhere in the chain
}
```

**Note:** `errors.Is` compares by pointer identity on the sentinel by default. Wrapping with `.New(cause)` creates a new `*AppError`, which would break `errors.Is` comparison — but `AppError` implements `Is()` (matching by `Code`), so `errors.Is(err, domain.ErrNotFound)` works correctly through the chain. Use `errors.As` when you need access to the full `AppError` value.

---

## Validation Error Formatting

Gin binding errors (`ShouldBindJSON` failure) return raw `validator/v10` error messages. Format them into field-level errors for API consumers.

```go
// internal/handler/validation.go
package handler

import (
    "errors"
    "fmt"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/go-playground/validator/v10"
)

// FieldError represents a single field validation failure.
type FieldError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

// validationErrors formats validator.ValidationErrors into field-level messages.
func validationErrors(err error) []FieldError {
    var ve validator.ValidationErrors
    if !errors.As(err, &ve) {
        return []FieldError{{Field: "request", Message: err.Error()}}
    }

    out := make([]FieldError, 0, len(ve))
    for _, fe := range ve {
        out = append(out, FieldError{
            Field:   fe.Field(),
            Message: fieldMessage(fe),
        })
    }
    return out
}

func fieldMessage(fe validator.FieldError) string {
    switch fe.Tag() {
    case "required":
        return "field is required"
    case "email":
        return "must be a valid email address"
    case "min":
        return fmt.Sprintf("must be at least %s characters", fe.Param())
    case "max":
        return fmt.Sprintf("must be at most %s characters", fe.Param())
    case "oneof":
        return fmt.Sprintf("must be one of: %s", fe.Param())
    default:
        return fmt.Sprintf("failed validation: %s", fe.Tag())
    }
}

// Usage in handler:
func (h *UserHandler) Create(c *gin.Context) {
    var req domain.CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{
            "error":  "validation failed",
            "fields": validationErrors(err),
        })
        return
    }
    // ...
}
```

Response body:
```json
{
  "error": "validation failed",
  "fields": [
    {"field": "Email", "message": "must be a valid email address"},
    {"field": "Password", "message": "must be at least 8 characters"}
  ]
}
```

---

## Panic Recovery Middleware

Use custom recovery to return JSON instead of Gin's default plain-text 500.

```go
// pkg/middleware/recovery.go
package middleware

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
)

// Recovery returns a Gin middleware that recovers from panics and logs them.
// Register with r.Use(middleware.Recovery(logger)) instead of gin.Recovery().
func Recovery(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if rec := recover(); rec != nil {
                logger.ErrorContext(c.Request.Context(), "panic recovered",
                    "panic", rec,
                    "path", c.FullPath(),
                    "method", c.Request.Method,
                )
                c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                    "error": "internal server error",
                })
            }
        }()
        c.Next()
    }
}
```

Register in main.go:

```go
r := gin.New()
r.Use(middleware.Logger(logger))
r.Use(middleware.Recovery(logger))
```

---

## Consistent JSON Error Format

All error responses must follow the same shape so clients can parse them predictably.

```go
// ErrorResponse is the canonical error body shape.
// Use gin.H for simple errors, this type when you need field-level errors.
type ErrorResponse struct {
    Error  string       `json:"error"`
    Fields []FieldError `json:"fields,omitempty"` // only for validation errors
}
```

Examples:

```json
// 400 binding error
{"error": "invalid JSON body"}

// 400 validation error (with fields)
{"error": "validation failed", "fields": [{"field": "Email", "message": "must be a valid email address"}]}

// 401 unauthorized
{"error": "unauthorized"}

// 404 not found
{"error": "resource not found"}

// 409 conflict
{"error": "resource already exists"}

// 500 internal
{"error": "internal server error"}
```

**Critical:** Never include internal error messages, stack traces, or database errors in 5xx responses. Log internally, return generic message to client.
