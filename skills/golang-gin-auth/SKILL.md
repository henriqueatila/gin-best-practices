---
name: golang-gin-auth
description: "Implement authentication and authorization in Go Gin APIs. Covers JWT middleware, login/register handlers, role-based access control (RBAC), token refresh, and protected routes. Use when adding auth, login, signup, JWT tokens, user sessions, permissions, or role checks to a Gin application."
license: MIT
metadata:
  author: henriqueatila
  version: "1.0.0"
---

# golang-gin-auth — Authentication & Authorization

Add JWT-based authentication and role-based access control to a Gin API. This skill covers the patterns you need for secure APIs: JWT middleware, login handler, token lifecycle, and RBAC.

## When to Use

- Adding JWT authentication to a Gin API
- Implementing login or registration endpoints
- Protecting routes with middleware
- Implementing role-based or permission-based access control (RBAC)
- Handling token refresh and revocation
- Getting the current user in any handler

## Dependencies

```bash
go get github.com/golang-jwt/jwt/v5
go get golang.org/x/crypto
```

## Claims Struct

Define custom JWT claims that embed `jwt.RegisteredClaims` (architectural recommendation — uses `github.com/golang-jwt/jwt/v5`):

```go
// internal/auth/claims.go
package auth

import "github.com/golang-jwt/jwt/v5"

// Claims holds the JWT payload. Embed RegisteredClaims for standard fields.
type Claims struct {
    jwt.RegisteredClaims
    UserID string   `json:"uid"`
    Email  string   `json:"email"`
    Role   string   `json:"role"`
    // Add Roles []string for multi-role support, TenantID for multi-tenancy
}
```

## Token Generation and Validation

```go
// internal/auth/token.go
package auth

import (
    "errors"
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

// TokenConfig holds secrets and TTLs. Load from environment, never hardcode.
type TokenConfig struct {
    AccessSecret  []byte
    RefreshSecret []byte
    AccessTTL     time.Duration // e.g. 15 * time.Minute
    RefreshTTL    time.Duration // e.g. 7 * 24 * time.Hour
}

// GenerateAccessToken creates a signed JWT access token for the given user.
// Architectural recommendation — not part of the Gin API.
func GenerateAccessToken(cfg TokenConfig, userID, email, role string) (string, error) {
    claims := Claims{
        RegisteredClaims: jwt.RegisteredClaims{
            Subject:   userID,
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(cfg.AccessTTL)),
        },
        UserID: userID,
        Email:  email,
        Role:   role,
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(cfg.AccessSecret)
    if err != nil {
        return "", fmt.Errorf("sign access token: %w", err)
    }
    return signed, nil
}

// GenerateRefreshToken creates a longer-lived refresh token (user ID only).
func GenerateRefreshToken(cfg TokenConfig, userID string) (string, error) {
    claims := jwt.RegisteredClaims{
        Subject:   userID,
        IssuedAt:  jwt.NewNumericDate(time.Now()),
        ExpiresAt: jwt.NewNumericDate(time.Now().Add(cfg.RefreshTTL)),
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(cfg.RefreshSecret)
    if err != nil {
        return "", fmt.Errorf("sign refresh token: %w", err)
    }
    return signed, nil
}

// ParseAccessToken validates and parses a JWT access token.
func ParseAccessToken(cfg TokenConfig, tokenStr string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (any, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return cfg.AccessSecret, nil
    })
    if err != nil {
        if errors.Is(err, jwt.ErrTokenExpired) {
            return nil, fmt.Errorf("token expired: %w", err)
        }
        return nil, fmt.Errorf("invalid token: %w", err)
    }
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid token claims")
    }
    return claims, nil
}
```

## JWT Middleware

Extracts the Bearer token, validates it, and injects claims into `gin.Context` via `c.Set`. Handlers read claims via `c.Get`.

```go
// pkg/middleware/auth.go
package middleware

import (
    "log/slog"
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
    "myapp/internal/auth"
)

const (
    ClaimsKey = "claims"   // key used for c.Set / c.Get
    UserIDKey  = "user_id" // convenience string key for c.GetString
)

// Auth returns a Gin middleware that validates JWT Bearer tokens.
// Aborts with 401 if the token is missing, malformed, or expired.
func Auth(cfg auth.TokenConfig, logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        header := c.GetHeader("Authorization")
        if header == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "authorization header required"})
            return
        }

        parts := strings.SplitN(header, " ", 2)
        if len(parts) != 2 || !strings.EqualFold(parts[0], "Bearer") {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid authorization header format"})
            return
        }

        claims, err := auth.ParseAccessToken(cfg, parts[1])
        if err != nil {
            logger.Warn("jwt validation failed", "error", err)
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid or expired token"})
            return
        }

        // Inject claims and convenience string keys for type-safe access
        c.Set(ClaimsKey, claims)
        c.Set(UserIDKey, claims.UserID)
        c.Next()
    }
}
```

## Login Handler

Validates credentials via `UserRepository`, then returns both tokens. Thin handler — no business logic beyond orchestration.

```go
// internal/handler/auth_handler.go
package handler

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "golang.org/x/crypto/bcrypt"
    "myapp/internal/auth"
    "myapp/internal/domain"
)

type AuthHandler struct {
    userRepo domain.UserRepository
    tokenCfg auth.TokenConfig
    logger   *slog.Logger
}

func NewAuthHandler(userRepo domain.UserRepository, cfg auth.TokenConfig, logger *slog.Logger) *AuthHandler {
    return &AuthHandler{userRepo: userRepo, tokenCfg: cfg, logger: logger}
}

type loginRequest struct {
    Email    string `json:"email"    binding:"required,email"`
    Password string `json:"password" binding:"required"`
}

type tokenResponse struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
}

func (h *AuthHandler) Login(c *gin.Context) {
    var req loginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.userRepo.GetByEmail(c.Request.Context(), req.Email)
    if err != nil {
        // Return generic message — don't leak whether email exists
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
        return
    }

    if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(req.Password)); err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
        return
    }

    accessToken, err := auth.GenerateAccessToken(h.tokenCfg, user.ID, user.Email, user.Role)
    if err != nil {
        h.logger.Error("failed to generate access token", "error", err, "user_id", user.ID)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    refreshToken, err := auth.GenerateRefreshToken(h.tokenCfg, user.ID)
    if err != nil {
        h.logger.Error("failed to generate refresh token", "error", err, "user_id", user.ID)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    c.JSON(http.StatusOK, tokenResponse{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
    })
}
```

## Protected Routes — Applying Auth Middleware

Wire the engine with `gin.New()` + explicit middleware, then apply `Auth` to route groups:

```go
// cmd/api/main.go — engine setup (always gin.New(), never gin.Default() in production)
r := gin.New()
r.Use(middleware.Logger(logger))
r.Use(middleware.Recovery(logger))
registerRoutes(r, authHandler, userHandler, tokenCfg, logger)
```

```go
// cmd/api/main.go (route registration)
func registerRoutes(r *gin.Engine, authHandler *handler.AuthHandler, userHandler *handler.UserHandler, cfg auth.TokenConfig, logger *slog.Logger) {
    r.GET("/health", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })

    api := r.Group("/api/v1")

    // Public routes — no auth
    public := api.Group("")
    {
        public.POST("/auth/login", authHandler.Login)
        public.POST("/auth/refresh", authHandler.Refresh)
        public.POST("/users", userHandler.Create) // registration
    }

    // Protected routes — JWT required
    protected := api.Group("")
    protected.Use(middleware.Auth(cfg, logger))
    {
        protected.GET("/users/:id", userHandler.GetByID)
        protected.PUT("/users/:id", userHandler.Update)
        protected.DELETE("/users/:id", userHandler.Delete)

        // Admin-only routes — add RBAC middleware
        admin := protected.Group("/admin")
        admin.Use(middleware.RequireRole("admin"))
        {
            admin.GET("/users", userHandler.List)
        }
    }
}
```

## Getting Current User in Handlers

```go
// Read claims injected by Auth middleware
func (h *UserHandler) GetMe(c *gin.Context) {
    // Option 1: type-safe string shortcut
    userID := c.GetString(middleware.UserIDKey)

    // Option 2: full claims object (for role, email, etc.)
    val, exists := c.Get(middleware.ClaimsKey)
    if !exists {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
        return
    }
    claims, ok := val.(*auth.Claims)
    if !ok {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    user, err := h.svc.GetByID(c.Request.Context(), userID)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "user": user,
        "role": claims.Role,
    })
}
```

## Reference Files

Load these when you need deeper detail:

- **[references/jwt-patterns.md](references/jwt-patterns.md)** — Access + refresh token architecture, token refresh endpoint, token blacklisting (Redis), RS256 vs HS256, custom claims, storage recommendations (httpOnly cookie vs localStorage), complete auth flow
- **[references/rbac.md](references/rbac.md)** — RequireRole/RequireAnyRole middleware, permission-based access, role hierarchy, multi-tenant authorization, resource-level authorization, complete RBAC example

## Cross-Skill References

- For handler patterns (ShouldBindJSON, error responses, route groups): see the **golang-gin-api** skill
- For `UserRepository` interface and `GetByEmail` implementation: see the **golang-gin-database** skill
- For testing JWT middleware and auth handlers: see the **golang-gin-testing** skill
- **golang-gin-clean-arch** → Architecture: where auth middleware fits (delivery layer only), DI patterns for auth services

## Official Docs

If this skill doesn't cover your use case, consult the [Gin documentation](https://gin-gonic.com/docs/), [golang-jwt GoDoc](https://pkg.go.dev/github.com/golang-jwt/jwt/v5), or [Gin GoDoc](https://pkg.go.dev/github.com/gin-gonic/gin).
