# rbac.md — Role-Based Access Control

Complete reference for RBAC and permission-based authorization in Gin APIs. Covers role middleware, permission checks, role hierarchy, multi-tenant auth, and resource-level authorization.

All patterns are architectural recommendations — they use the `Claims` struct from the gin-auth SKILL.md and Gin's `c.Set`/`c.Get` for context propagation.

## Table of Contents

1. [Role-Based Middleware](#role-based-middleware)
2. [Permission-Based Middleware](#permission-based-middleware)
3. [Role Hierarchy](#role-hierarchy)
4. [Multi-Tenant Authorization](#multi-tenant-authorization)
5. [Resource-Level Authorization](#resource-level-authorization)
6. [Admin Impersonation](#admin-impersonation)
7. [Complete RBAC Example](#complete-rbac-example)

---

## Role-Based Middleware

`RequireRole` and `RequireAnyRole` sit after `Auth` middleware in the chain. `Auth` validates the JWT and injects claims; RBAC middleware reads the role from those claims.

```go
// pkg/middleware/rbac.go
package middleware

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "myapp/internal/auth"
)

// RequireRole aborts with 403 if the authenticated user's role does not match.
// Must be used after Auth middleware.
func RequireRole(role string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        if claims.Role != role {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "insufficient role"})
            return
        }
        c.Next()
    }
}

// RequireAnyRole aborts with 403 if the user's role is not in the allowed set.
func RequireAnyRole(roles ...string) gin.HandlerFunc {
    allowed := make(map[string]struct{}, len(roles))
    for _, r := range roles {
        allowed[r] = struct{}{}
    }
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        if _, ok := allowed[claims.Role]; !ok {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "insufficient role"})
            return
        }
        c.Next()
    }
}

// claimsFromContext is a helper to safely extract *auth.Claims from gin.Context.
func claimsFromContext(c *gin.Context) *auth.Claims {
    val, exists := c.Get(ClaimsKey)
    if !exists {
        return nil
    }
    claims, _ := val.(*auth.Claims)
    return claims
}
```

**Usage in route registration:**

```go
protected := api.Group("")
protected.Use(middleware.Auth(tokenCfg, logger))
{
    // Any authenticated user
    protected.GET("/users/me", userHandler.GetMe)

    // Moderators and admins
    modGroup := protected.Group("")
    modGroup.Use(middleware.RequireAnyRole("admin", "moderator"))
    {
        modGroup.PUT("/posts/:id/hide", postHandler.Hide)
    }

    // Admins only
    adminGroup := protected.Group("/admin")
    adminGroup.Use(middleware.RequireRole("admin"))
    {
        adminGroup.GET("/users", userHandler.List)
        adminGroup.DELETE("/users/:id", userHandler.Delete)
    }
}
```

---

## Permission-Based Middleware

For fine-grained control beyond roles, embed permissions in the JWT claims or check them against a database.

**Option A — permissions in JWT claims** (fast, no DB lookup per request):

```go
// internal/auth/claims.go (extended)
package auth

import "github.com/golang-jwt/jwt/v5"

type Claims struct {
    jwt.RegisteredClaims
    UserID      string   `json:"uid"`
    Email       string   `json:"email"`
    Role        string   `json:"role"`
    Permissions []string `json:"perms"` // e.g. ["posts:write", "users:read"]
}
```

```go
// pkg/middleware/rbac.go (additional function)

// RequirePermission aborts with 403 if the user does not have the required permission.
// Permission format convention: "resource:action" (e.g. "users:delete").
func RequirePermission(permission string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        for _, p := range claims.Permissions {
            if p == permission {
                c.Next()
                return
            }
        }
        c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "permission denied"})
    }
}
```

**Option B — DB lookup per request** (always up-to-date, higher latency):

```go
// pkg/middleware/rbac.go
func RequirePermissionDB(permSvc PermissionService, permission string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }

        ok, err := permSvc.HasPermission(c.Request.Context(), claims.UserID, permission)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
            return
        }
        if !ok {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "permission denied"})
            return
        }
        c.Next()
    }
}

// PermissionService is an interface — define at the consumer, implement in the service layer.
type PermissionService interface {
    HasPermission(ctx context.Context, userID, permission string) (bool, error)
}
```

**Trade-off:** JWT permissions are fast but stale (user retains permissions until token expires). DB lookup is always current but adds latency. Cache the DB result with a short TTL (e.g. 30s in Redis) as a middle ground.

---

## Role Hierarchy

Encode hierarchy as a map — higher-ranked roles inherit lower-ranked permissions.

```go
// internal/auth/roles.go
package auth

// roleRank maps role name to numeric rank. Higher = more privileged.
var roleRank = map[string]int{
    "guest":     0,
    "user":      1,
    "moderator": 2,
    "admin":     3,
    "superadmin": 4,
}

// HasRoleAtLeast returns true if userRole has rank >= requiredRole.
// Returns false for any unrecognized role — unknown roles are denied, not defaulted to guest.
func HasRoleAtLeast(userRole, requiredRole string) bool {
    ur, ok1 := roleRank[userRole]
    rr, ok2 := roleRank[requiredRole]
    return ok1 && ok2 && ur >= rr
}
```

```go
// pkg/middleware/rbac.go

// RequireMinRole allows any role with rank >= minRole (e.g. "moderator" also allows "admin").
func RequireMinRole(minRole string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        if !auth.HasRoleAtLeast(claims.Role, minRole) {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
                "error": "insufficient privileges",
            })
            return
        }
        c.Next()
    }
}
```

---

## Multi-Tenant Authorization

In multi-tenant systems, users belong to a tenant. Requests must be validated both for authentication AND tenant membership.

**Add TenantID to claims:**

```go
// internal/auth/claims.go
type Claims struct {
    jwt.RegisteredClaims
    UserID   string `json:"uid"`
    Email    string `json:"email"`
    Role     string `json:"role"`
    TenantID string `json:"tid"` // tenant identifier
}
```

**Tenant middleware — injects TenantID from claims into context:**

```go
// pkg/middleware/tenant.go
package middleware

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

const TenantIDKey = "tenant_id"

// RequireTenant ensures requests carry a valid tenant context.
// Must be used after Auth middleware.
func RequireTenant() gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil || claims.TenantID == "" {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "tenant context required"})
            return
        }
        c.Set(TenantIDKey, claims.TenantID)
        c.Next()
    }
}
```

**Enforce tenant isolation in repositories:**

```go
// internal/repository/post_repository_gorm.go

func (r *gormPostRepository) GetByID(ctx context.Context, tenantID, postID string) (*domain.Post, error) {
    var m PostModel
    // Always scope queries to the tenant — prevents cross-tenant data leaks
    err := r.db.WithContext(ctx).
        Where("tenant_id = ? AND id = ?", tenantID, postID).
        First(&m).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, domain.ErrNotFound
        }
        return nil, domain.ErrInternal
    }
    return m.ToDomain(), nil
}
```

**Handler reads tenant from context:**

```go
func (h *PostHandler) GetByID(c *gin.Context) {
    tenantID := c.GetString(middleware.TenantIDKey)

    type uriParams struct {
        ID string `uri:"id" binding:"required"`
    }
    var params uriParams
    if err := c.ShouldBindURI(&params); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request"})
        return
    }

    post, err := h.svc.GetByID(c.Request.Context(), tenantID, params.ID)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, post)
}
```

---

## Resource-Level Authorization

> **Security note:** In production, never expose raw `err.Error()` to clients. Return generic messages and log the error server-side. See **golang-gin-clean-arch** error handling patterns.

Verify the authenticated user owns or has access to the specific resource — not just the right role.

```go
// internal/handler/user_handler.go

// Update allows users to edit their own profile, or admins to edit any profile.
func (h *UserHandler) Update(c *gin.Context) {
    type uriParams struct {
        ID string `uri:"id" binding:"required"`
    }
    var params uriParams
    if err := c.ShouldBindURI(&params); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request"})
        return
    }

    claims := claimsFromContext(c)
    if claims == nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
        return
    }

    // Resource-level check: only the owner or an admin may update
    if claims.UserID != params.ID && claims.Role != "admin" {
        c.JSON(http.StatusForbidden, gin.H{"error": "cannot modify another user's profile"})
        return
    }

    var req domain.UpdateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    user, err := h.svc.Update(c.Request.Context(), params.ID, req)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, user)
}
```

**Why in the handler, not middleware?** Resource ownership depends on business logic (the specific resource ID). Middleware runs before the route parameters are bound to a domain object — the handler is the right layer for this check.

---

## Admin Impersonation

Allows admins to act as another user for support or debugging. Inject an `ImpersonatedBy` field so audit logs capture both identities.

```go
// internal/auth/claims.go (extended)
type Claims struct {
    jwt.RegisteredClaims
    UserID         string `json:"uid"`
    Email          string `json:"email"`
    Role           string `json:"role"`
    ImpersonatedBy string `json:"imp,omitempty"` // set when admin impersonates
}
```

```go
// internal/handler/admin_handler.go

type impersonateRequest struct {
    TargetUserID string `json:"target_user_id" binding:"required"`
}

// Impersonate generates a short-lived access token for a target user.
// Only admins may call this endpoint.
func (h *AdminHandler) Impersonate(c *gin.Context) {
    adminClaims := claimsFromContext(c)
    if adminClaims == nil || adminClaims.Role != "admin" {
        c.JSON(http.StatusForbidden, gin.H{"error": "admin access required"})
        return
    }

    var req impersonateRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    target, err := h.userRepo.GetByID(c.Request.Context(), req.TargetUserID)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    // Prevent admin-to-admin privilege escalation
    if target.Role == "admin" || target.Role == "superadmin" {
        c.JSON(http.StatusForbidden, gin.H{"error": "cannot impersonate admin users"})
        return
    }

    // Build impersonation token: short TTL, records impersonating admin
    claims := auth.Claims{
        RegisteredClaims: jwt.RegisteredClaims{
            Subject:   target.ID,
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(1 * time.Hour)), // short TTL
        },
        UserID:         target.ID,
        Email:          target.Email,
        Role:           target.Role,
        ImpersonatedBy: adminClaims.UserID,
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(h.tokenCfg.AccessSecret)
    if err != nil {
        h.logger.Error("impersonate: failed to sign token", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    h.logger.Info("admin impersonation", "admin_id", adminClaims.UserID, "target_id", target.ID)
    c.JSON(http.StatusOK, gin.H{"access_token": signed})
}
```

**Critical:** Log all impersonation events. Restrict the impersonation endpoint to `superadmin` role if you have a role hierarchy. Never allow impersonating another admin.

---

## Complete RBAC Example

Full wiring from routes to middleware to handler, using all the patterns above:

```go
// cmd/api/main.go — route registration with RBAC
func registerRoutes(
    r *gin.Engine,
    authHandler *handler.AuthHandler,
    userHandler *handler.UserHandler,
    postHandler *handler.PostHandler,
    adminHandler *handler.AdminHandler,
    tokenCfg auth.TokenConfig,
    logger *slog.Logger,
) {
    api := r.Group("/api/v1")

    // Public endpoints — no auth required
    api.POST("/auth/login", authHandler.Login)
    api.POST("/auth/refresh", authHandler.Refresh)
    api.POST("/users", userHandler.Create)

    // Authenticated endpoints
    authed := api.Group("")
    authed.Use(middleware.Auth(tokenCfg, logger))
    {
        authed.POST("/auth/logout", authHandler.Logout)

        // Self-service — any authenticated user
        authed.GET("/users/me", userHandler.GetMe)
        authed.PUT("/users/:id", userHandler.Update) // resource-level check inside handler

        // Posts — user and above (role hierarchy)
        posts := authed.Group("/posts")
        posts.Use(middleware.RequireMinRole("user"))
        {
            posts.GET("", postHandler.List)
            posts.GET("/:id", postHandler.GetByID)
            posts.POST("", postHandler.Create)
        }

        // Moderation — moderator and above
        moderation := authed.Group("/moderation")
        moderation.Use(middleware.RequireMinRole("moderator"))
        {
            moderation.PUT("/posts/:id/hide", postHandler.Hide)
            moderation.DELETE("/posts/:id", postHandler.Delete)
        }

        // Admin panel — admin role only
        admin := authed.Group("/admin")
        admin.Use(middleware.RequireRole("admin"))
        {
            admin.GET("/users", userHandler.List)
            admin.DELETE("/users/:id", userHandler.Delete)
            admin.POST("/impersonate", adminHandler.Impersonate)
        }
    }
}
```

**Middleware execution order for a protected admin route:**

```
Request → Auth (validate JWT, inject claims)
        → RequireRole("admin") (check claims.Role)
        → Handler (resource-level check if needed)
        → Response
```

**Key principles:**
- Auth middleware is always first in the protected group
- Role/permission middleware comes immediately after Auth
- Resource-level checks (owner == caller) belong in the handler, not middleware
- Use `RequireMinRole` for hierarchy, `RequireAnyRole` for explicit sets, `RequireRole` for exact match
- Always log authorization failures with enough context to audit (user ID, resource, action)
