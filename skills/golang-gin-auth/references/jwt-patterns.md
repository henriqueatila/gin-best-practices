# jwt-patterns.md — JWT Token Architecture

Complete reference for JWT access/refresh token patterns in Gin APIs. Covers token lifecycle, refresh endpoint, blacklisting, storage, and algorithm choices.

All patterns use `github.com/golang-jwt/jwt/v5`. These are architectural recommendations, not Gin API.

## Table of Contents

1. [Access + Refresh Token Architecture](#access--refresh-token-architecture)
2. [Custom Claims](#custom-claims)
3. [Token Refresh Endpoint](#token-refresh-endpoint)
4. [Token Blacklisting (Redis)](#token-blacklisting-redis)
5. [Storage Recommendations](#storage-recommendations)
6. [RS256 vs HS256](#rs256-vs-hs256)
7. [Complete Auth Flow Example](#complete-auth-flow-example)

---

## Access + Refresh Token Architecture

Two-token strategy separates concerns: short-lived access tokens reduce attack surface; long-lived refresh tokens enable session continuity without re-login.

```
┌──────────┐   POST /auth/login    ┌───────────┐
│  Client  │──────────────────────▶│  Gin API  │
│          │◀──────────────────────│           │
│          │  {access, refresh}    └───────────┘
│          │
│          │   GET /api/v1/users   ┌───────────┐
│          │   Authorization: Bearer <access>
│          │──────────────────────▶│  Gin API  │
│          │◀──────────────────────│           │
│          │   200 OK              └───────────┘
│          │
│          │   POST /auth/refresh  ┌───────────┐
│          │   {refresh_token}     │           │
│          │──────────────────────▶│  Gin API  │
│          │◀──────────────────────│           │
│          │   {new_access}        └───────────┘
└──────────┘
```

| Token         | TTL           | Payload         | Stored where       |
|---------------|---------------|-----------------|--------------------|
| Access token  | 15 minutes    | UserID, Role    | Memory / header    |
| Refresh token | 7–30 days     | UserID only     | httpOnly cookie     |

**Why short-lived access tokens?** If stolen, they expire quickly. The refresh token grants new access tokens — revoking a refresh token immediately locks out the session.

---

## Custom Claims

Embed `jwt.RegisteredClaims` for standard fields. Add only what handlers need — avoid putting large objects in the token.

```go
// internal/auth/claims.go
package auth

import "github.com/golang-jwt/jwt/v5"

// Claims is the JWT payload for access tokens.
type Claims struct {
    jwt.RegisteredClaims            // sub, iat, exp, iss, aud
    UserID   string   `json:"uid"`
    Email    string   `json:"email"`
    Role     string   `json:"role"`
    // For multi-role: Roles    []string `json:"roles"`
    // For multi-tenant: TenantID string `json:"tid"`
}

// RefreshClaims is the minimal payload for refresh tokens.
// Only the subject (UserID) is needed — refresh doesn't need role/email.
type RefreshClaims struct {
    jwt.RegisteredClaims
    TokenID string `json:"jti"` // for blacklisting: store this ID in Redis on logout
}
```

**Why separate RefreshClaims?** Refresh tokens are long-lived — embedding extra data increases exposure. The `jti` (JWT ID) enables per-token revocation without invalidating all sessions.

---

## Token Refresh Endpoint

Exchange a valid refresh token for a new access token. Do not rotate the refresh token on every call (causes logout on parallel requests); rotate on logout or suspicious activity.

```go
// internal/handler/auth_handler.go (Refresh method)
package handler

import (
    "fmt"
    "log/slog"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v5"
    "myapp/internal/auth"
    "myapp/internal/domain"
)

type refreshRequest struct {
    RefreshToken string `json:"refresh_token" binding:"required"`
}

func (h *AuthHandler) Refresh(c *gin.Context) {
    var req refreshRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Parse refresh token with its own secret
    token, err := jwt.ParseWithClaims(req.RefreshToken, &auth.RefreshClaims{}, func(t *jwt.Token) (any, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return h.tokenCfg.RefreshSecret, nil
    })
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid or expired refresh token"})
        return
    }

    rc, ok := token.Claims.(*auth.RefreshClaims)
    if !ok || !token.Valid {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid refresh token claims"})
        return
    }

    // Re-fetch user to ensure they are still active
    user, err := h.userRepo.GetByID(c.Request.Context(), rc.Subject)
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "user not found"})
        return
    }

    accessToken, err := auth.GenerateAccessToken(h.tokenCfg, user.ID, user.Email, user.Role)
    if err != nil {
        h.logger.Error("failed to generate access token on refresh", "error", err, "user_id", user.ID)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    // Also issue a new refresh token (optional rotation strategy)
    newRefresh, err := auth.GenerateRefreshToken(h.tokenCfg, user.ID)
    if err != nil {
        h.logger.Error("failed to generate refresh token on refresh", "error", err, "user_id", user.ID)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "access_token":  accessToken,
        "refresh_token": newRefresh,
        "expires_in":    int(h.tokenCfg.AccessTTL / time.Second),
    })
}
```

---

## Token Blacklisting (Redis)

Tokens are stateless — you can't "delete" one. Blacklisting stores revoked token IDs in Redis with TTL matching the token's remaining lifetime. On every request, the middleware checks Redis before accepting the token.

```go
// internal/auth/blacklist.go
package auth

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

// Blacklist manages token revocation via Redis.
// Architectural recommendation — not part of the Gin API.
type Blacklist struct {
    rdb    *redis.Client
    prefix string
}

func NewBlacklist(rdb *redis.Client) *Blacklist {
    return &Blacklist{rdb: rdb, prefix: "jwt:revoked:"}
}

// Revoke adds a token ID to the blacklist until its expiry.
func (b *Blacklist) Revoke(ctx context.Context, tokenID string, expiry time.Time) error {
    ttl := time.Until(expiry)
    if ttl <= 0 {
        return nil // already expired — no need to blacklist
    }
    key := b.prefix + tokenID
    if err := b.rdb.Set(ctx, key, "1", ttl).Err(); err != nil {
        return fmt.Errorf("blacklist.Revoke: %w", err)
    }
    return nil
}

// IsRevoked returns true if the token ID has been revoked.
func (b *Blacklist) IsRevoked(ctx context.Context, tokenID string) (bool, error) {
    key := b.prefix + tokenID
    val, err := b.rdb.Exists(ctx, key).Result()
    if err != nil {
        return false, fmt.Errorf("blacklist.IsRevoked: %w", err)
    }
    return val > 0, nil
}
```

**Middleware integration:** After `ParseAccessToken`, check `blacklist.IsRevoked(ctx, claims.ID)` before calling `c.Next()`. The `claims.ID` maps to the `jti` (JWT ID) field in `RegisteredClaims`.

**Logout endpoint pattern:**
```go
func (h *AuthHandler) Logout(c *gin.Context) {
    val, exists := c.Get(middleware.ClaimsKey)
    if !exists {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
        return
    }
    claims := val.(*auth.Claims)

    expiry := claims.ExpiresAt.Time
    if err := h.blacklist.Revoke(c.Request.Context(), claims.ID, expiry); err != nil {
        h.logger.Error("failed to revoke token", "error", err, "jti", claims.ID)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    c.JSON(http.StatusOK, gin.H{"message": "logged out"})
}
```

---

## Storage Recommendations

Where the client stores tokens affects the attack surface:

| Storage          | XSS Risk | CSRF Risk | Notes                                     |
|------------------|----------|-----------|-------------------------------------------|
| httpOnly cookie  | Low      | Medium    | Use `SameSite=Strict` + CSRF token to mitigate |
| localStorage     | High     | Low       | JavaScript-accessible — XSS can steal tokens |
| sessionStorage   | High     | Low       | Same as localStorage, cleared on tab close |
| Memory (JS var)  | Low      | Low       | Lost on refresh — needs silent refresh flow |

**Recommended:** httpOnly, Secure cookie for refresh token; short-lived access token in memory or Authorization header.

Setting a cookie in Gin:
```go
// After successful login, set refresh token as httpOnly cookie
c.SetCookie(
    "refresh_token",      // name
    refreshToken,         // value
    int(7*24*time.Hour/time.Second), // maxAge in seconds
    "/auth",              // path — restrict to /auth/* endpoints
    "",                   // domain (empty = current host)
    true,                 // secure (HTTPS only)
    true,                 // httpOnly (not accessible by JS)
)
```

**Reading the cookie in the refresh endpoint:**
```go
refreshToken, err := c.Cookie("refresh_token")
if err != nil {
    c.JSON(http.StatusUnauthorized, gin.H{"error": "refresh token not found"})
    return
}
```

---

## RS256 vs HS256

| Aspect          | HS256 (HMAC-SHA256)         | RS256 (RSA-SHA256)                  |
|-----------------|-----------------------------|-------------------------------------|
| Keys            | One shared secret           | Private key (sign) + Public key (verify) |
| Verification    | Needs the secret            | Needs only the public key           |
| Use case        | Single service / monolith   | Microservices, third-party verification |
| Key management  | Simple                      | More complex (cert rotation)        |

**HS256 setup** (shown in SKILL.md) — suitable for most single-service APIs.

**RS256 setup** — use when multiple services or external clients need to verify tokens without the signing secret:

```go
// internal/auth/token_rsa.go
package auth

import (
    "crypto/rsa"
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

// GenerateAccessTokenRS256 signs with RSA private key.
func GenerateAccessTokenRS256(privateKey *rsa.PrivateKey, userID, email, role string, ttl time.Duration) (string, error) {
    claims := Claims{
        RegisteredClaims: jwt.RegisteredClaims{
            Subject:   userID,
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(ttl)),
        },
        UserID: userID,
        Email:  email,
        Role:   role,
    }
    token := jwt.NewWithClaims(jwt.SigningMethodRS256, claims)
    signed, err := token.SignedString(privateKey)
    if err != nil {
        return "", fmt.Errorf("sign RS256 token: %w", err)
    }
    return signed, nil
}

// ParseAccessTokenRS256 verifies with RSA public key (safe to distribute).
func ParseAccessTokenRS256(publicKey *rsa.PublicKey, tokenStr string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (any, error) {
        if _, ok := t.Method.(*jwt.SigningMethodRSA); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return publicKey, nil
    })
    if err != nil {
        return nil, fmt.Errorf("parse RS256 token: %w", err)
    }
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid token claims")
    }
    return claims, nil
}
```

Load RSA keys from PEM files or environment at startup — never embed keys in source code.

---

## Complete Auth Flow Example

End-to-end wiring of auth routes with all patterns:

```go
// cmd/api/main.go — auth wiring
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
    "github.com/redis/go-redis/v9"
    "myapp/internal/auth"
    "myapp/internal/handler"
    "myapp/internal/repository"
    "myapp/internal/service"
    "myapp/pkg/middleware"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    // Load config from environment — never hardcode secrets
    tokenCfg := auth.TokenConfig{
        AccessSecret:  []byte(os.Getenv("JWT_ACCESS_SECRET")),
        RefreshSecret: []byte(os.Getenv("JWT_REFRESH_SECRET")),
        AccessTTL:     15 * time.Minute,
        RefreshTTL:    7 * 24 * time.Hour,
    }

    // Redis for token blacklisting
    rdb := redis.NewClient(&redis.Options{Addr: os.Getenv("REDIS_ADDR")})
    blacklist := auth.NewBlacklist(rdb)

    // DB and repository wiring (see gin-database skill)
    db, err := repository.NewGORMDB(repository.Config{DSN: os.Getenv("DATABASE_URL")}, logger)
    if err != nil {
        logger.Error("database connection failed", "error", err)
        os.Exit(1)
    }
    userRepo := repository.NewUserRepository(db)

    // Handlers
    authHandler := handler.NewAuthHandler(userRepo, tokenCfg, blacklist, logger)
    userSvc := service.NewUserService(userRepo, logger)
    userHandler := handler.NewUserHandler(userSvc, logger)

    r := gin.New()
    r.Use(middleware.Logger(logger))
    r.Use(middleware.Recovery(logger))

    api := r.Group("/api/v1")

    // Public
    api.POST("/auth/login", authHandler.Login)
    api.POST("/auth/refresh", authHandler.Refresh)
    api.POST("/users", userHandler.Create)

    // Protected — Auth middleware validates JWT and injects claims
    protected := api.Group("")
    protected.Use(middleware.Auth(tokenCfg, logger))
    {
        protected.POST("/auth/logout", authHandler.Logout)
        protected.GET("/users/me", userHandler.GetMe)
        protected.GET("/users/:id", userHandler.GetByID)

        // Admin only
        admin := protected.Group("/admin")
        admin.Use(middleware.RequireRole("admin"))
        {
            admin.GET("/users", userHandler.List)
            admin.DELETE("/users/:id", userHandler.Delete)
        }
    }

    srv := &http.Server{
        Addr:         os.Getenv("PORT"),
        Handler:      r,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 10 * time.Second,
    }
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            logger.Error("server failed", "error", err)
            os.Exit(1)
        }
    }()
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        logger.Error("graceful shutdown failed", "error", err)
    }
}
```

**Sequence for a protected request:**

1. Client sends `Authorization: Bearer <access_token>`
2. `Auth` middleware extracts and validates the token
3. Middleware checks `blacklist.IsRevoked(ctx, claims.ID)` → 401 if revoked
4. Claims injected via `c.Set(ClaimsKey, claims)` and `c.Set(UserIDKey, claims.UserID)`
5. Handler calls `c.GetString(UserIDKey)` or `c.Get(ClaimsKey)` to read identity
6. Handler passes `c.Request.Context()` to all downstream service/repository calls
