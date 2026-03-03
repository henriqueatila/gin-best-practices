# Routing Reference

This file covers advanced routing patterns for Gin: route groups and nesting, API versioning, path and query parameter binding, wildcard routes, NoRoute handler, custom validators, request size limits, and multipart file upload handling.

## Table of Contents

1. [Route Groups & Nesting](#route-groups--nesting)
2. [API Versioning](#api-versioning)
3. [Path Parameter Patterns](#path-parameter-patterns)
4. [Query Parameter Binding with Pagination](#query-parameter-binding-with-pagination)
5. [Wildcard Routes & NoRoute Handler](#wildcard-routes--noroute-handler)
6. [Custom Validators](#custom-validators)
7. [Request Size Limits](#request-size-limits)
8. [Multipart File Upload](#multipart-file-upload)

---

## Route Groups & Nesting

Groups share a path prefix and optional middleware. Use `{}` blocks for readability — they are cosmetic only (Go scope, not Gin scope).

```go
func registerRoutes(r *gin.Engine, h *handler.Handlers) {
    // Health outside versioning — no auth, no version prefix
    r.GET("/health", h.Health.Check)

    api := r.Group("/api")
    {
        v1 := api.Group("/v1")
        {
            // Public: no middleware beyond global
            auth := v1.Group("/auth")
            {
                auth.POST("/login", h.Auth.Login)
                auth.POST("/register", h.Auth.Register)
                auth.POST("/refresh", h.Auth.Refresh)
            }

            // Protected: JWT required
            users := v1.Group("/users")
            users.Use(middleware.Auth(tokenCfg, logger))
            {
                users.GET("", h.User.List)
                users.POST("", h.User.Create)
                users.GET("/:id", h.User.GetByID)
                users.PUT("/:id", h.User.Update)
                users.DELETE("/:id", h.User.Delete)
            }

            // Admin: JWT + role check
            admin := v1.Group("/admin")
            admin.Use(middleware.Auth(tokenCfg, logger), middleware.RequireRole("admin"))
            {
                admin.GET("/users", h.Admin.ListAllUsers)
                admin.DELETE("/users/:id", h.Admin.DeleteUser)
            }
        }
    }
}
```

**Why group nesting:** Each level applies its middleware to all routes below. This avoids repeating `middleware.Auth(tokenCfg, logger)` on every route and makes the security model explicit in the structure.

---

## API Versioning

### URL Path Versioning (recommended)

Path versioning is explicit, cache-friendly, and easy to route at the load-balancer level.

```go
v1 := r.Group("/api/v1")
v2 := r.Group("/api/v2")

// v1 handler
v1.GET("/users", listUsersV1)

// v2 handler — different response shape
v2.GET("/users", listUsersV2)
```

### Header-Based Versioning

Use when you want a stable URL but need to evolve the API contract.

```go
// pkg/middleware/version.go
func APIVersion() gin.HandlerFunc {
    return func(c *gin.Context) {
        version := c.GetHeader("API-Version")
        if version == "" {
            version = "v1" // default
        }
        c.Set("api_version", version)
        c.Next()
    }
}

// Handler reads version from context
func listUsers(c *gin.Context) {
    version := c.GetString("api_version")
    switch version {
    case "v2":
        // return v2 shape
    default:
        // return v1 shape
    }
}
```

```go
// Register with single path
r.Use(middleware.APIVersion())
r.GET("/api/users", listUsers)
```

---

## Path Parameter Patterns

> **Security note:** In production, never expose raw `err.Error()` to clients. Return generic messages and log the error server-side. See **golang-gin-clean-arch** error handling patterns.

Use `ShouldBindURI` with a struct to bind multiple path parameters at once.

```go
// Single parameter
r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id") // returns "/uuid-here" — note leading slash for wildcard, not for :param
    // For :id, c.Param returns the value without slash
})

// Multiple parameters with struct binding
r.GET("/orgs/:orgID/users/:userID", func(c *gin.Context) {
    type params struct {
        OrgID  string `uri:"orgID"  binding:"required"`
        UserID string `uri:"userID" binding:"required"`
    }
    var p params
    if err := c.ShouldBindURI(&p); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    // p.OrgID, p.UserID are bound and validated
})
```

**Critical:** URI parameter names in the `uri:""` tag must match the `:name` in the route path exactly.

---

## Query Parameter Binding with Pagination

Bind all query parameters at once using `ShouldBindQuery`. Define defaults via struct field initialisation before binding.

```go
// internal/domain/pagination.go
package domain

// ListOptions is the standard pagination + filter struct.
type ListOptions struct {
    Page   int    `form:"page"   binding:"min=1"`
    Limit  int    `form:"limit"  binding:"min=1,max=100"`
    Sort   string `form:"sort"   binding:"omitempty,oneof=created_at updated_at name"`
    Order  string `form:"order"  binding:"omitempty,oneof=asc desc"`
    Search string `form:"search" binding:"omitempty,max=100"`
    Role   string `form:"role"   binding:"omitempty,oneof=admin user"`
}

func (o *ListOptions) SetDefaults() {
    if o.Page == 0 {
        o.Page = 1
    }
    if o.Limit == 0 {
        o.Limit = 20
    }
    if o.Sort == "" {
        o.Sort = "created_at"
    }
    if o.Order == "" {
        o.Order = "desc"
    }
}
```

```go
// Handler
func (h *UserHandler) List(c *gin.Context) {
    var opts domain.ListOptions
    opts.SetDefaults()

    if err := c.ShouldBindQuery(&opts); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    users, total, err := h.svc.List(c.Request.Context(), opts)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "data":  users,
        "total": total,
        "page":  opts.Page,
        "limit": opts.Limit,
    })
}
```

---

## Wildcard Routes & NoRoute Handler

### Wildcard Parameters

`*param` matches everything after the prefix, including slashes.

```go
// Matches: /files/docs/report.pdf, /files/images/logo.png
r.GET("/files/*filepath", func(c *gin.Context) {
    filepath := c.Param("filepath") // includes leading slash: "/docs/report.pdf"
    // Serve file from storage...
})
```

### NoRoute Handler (Custom 404)

Replace Gin's default 404 with a JSON response.

```go
r.NoRoute(func(c *gin.Context) {
    c.JSON(http.StatusNotFound, gin.H{
        "error": "route not found",
        "path":  c.Request.URL.Path,
    })
})

// Required — NoMethod handler only fires when this is true (default: false)
r.HandleMethodNotAllowed = true

r.NoMethod(func(c *gin.Context) {
    c.JSON(http.StatusMethodNotAllowed, gin.H{
        "error":  "method not allowed",
        "method": c.Request.Method,
    })
})
```

---

## Custom Validators

Register custom validators via the `go-playground/validator/v10` engine. Do this once at startup.

```go
// pkg/validation/validators.go
package validation

import (
    "fmt"
    "time"

    "github.com/gin-gonic/gin/binding"
    "github.com/go-playground/validator/v10"
    "github.com/google/uuid"
)

func Register() error {
    v, ok := binding.Validator.Engine().(*validator.Validate)
    if !ok {
        return fmt.Errorf("unexpected validator type")
    }

    // Custom: validate that string is a valid UUID
    if err := v.RegisterValidation("uuid", validateUUID); err != nil {
        return fmt.Errorf("register uuid validator: %w", err)
    }

    // Custom: validate that int is a valid Unix timestamp (not in the past)
    if err := v.RegisterValidation("future_ts", validateFutureTimestamp); err != nil {
        return fmt.Errorf("register future_ts validator: %w", err)
    }

    return nil
}

func validateUUID(fl validator.FieldLevel) bool {
    val := fl.Field().String()
    _, err := uuid.Parse(val)
    return err == nil
}

func validateFutureTimestamp(fl validator.FieldLevel) bool {
    ts := fl.Field().Int()
    return time.Unix(ts, 0).After(time.Now())
}
```

Register in main.go before starting the server:

```go
if err := validation.Register(); err != nil {
    logger.Error("failed to register validators", "error", err)
    os.Exit(1)
}
```

Usage in request struct:

```go
type ScheduleRequest struct {
    UserID    string `json:"user_id"    binding:"required,uuid"`
    StartTime int64  `json:"start_time" binding:"required,future_ts"`
}
```

---

## Request Size Limits

Limit request body size to protect against large payload attacks.

```go
// pkg/middleware/limits.go
package middleware

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

// RequestSizeLimit returns middleware that limits request body size.
// maxBytes: maximum allowed body size in bytes.
func RequestSizeLimit(maxBytes int64) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, maxBytes)
        c.Next()
    }
}
```

```go
// Apply globally (1 MB limit for all routes)
r.Use(middleware.RequestSizeLimit(1 << 20)) // 1 MB

// Or per-route (10 MB for file upload route)
r.POST("/upload", middleware.RequestSizeLimit(10<<20), uploadHandler)
```

For multipart forms, also configure `engine.MaxMultipartMemory`:

```go
r := gin.New()
r.MaxMultipartMemory = 8 << 20 // 8 MB in-memory; rest spills to disk
```

---

## Multipart File Upload

```go
// internal/handler/upload_handler.go
package handler

import (
    "fmt"
    "net/http"
    "path/filepath"

    "github.com/gin-gonic/gin"
)

func (h *UploadHandler) UploadAvatar(c *gin.Context) {
    // ShouldBindURI for any path params first
    type uriParams struct {
        UserID string `uri:"id" binding:"required"`
    }
    var p uriParams
    if err := c.ShouldBindURI(&p); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Get single file
    file, err := c.FormFile("avatar")
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "avatar file required"})
        return
    }

    // Validate file type by extension
    ext := filepath.Ext(file.Filename)
    allowed := map[string]bool{".jpg": true, ".jpeg": true, ".png": true, ".webp": true}
    if !allowed[ext] {
        c.JSON(http.StatusBadRequest, gin.H{"error": "only jpg, png, webp files allowed"})
        return
    }

    // Validate file size (2 MB max)
    if file.Size > 2<<20 {
        c.JSON(http.StatusBadRequest, gin.H{"error": "file must be under 2 MB"})
        return
    }

    // Sanitize path: strip directory components to prevent path traversal
    safeID := filepath.Base(p.UserID)
    dst := filepath.Join("uploads/avatars", safeID+ext)
    if err := c.SaveUploadedFile(file, dst); err != nil {
        handleServiceError(c, domain.ErrInternal.New(fmt.Errorf("save file: %w", err)), h.logger)
        return
    }

    c.JSON(http.StatusOK, gin.H{"path": dst})
}

// Multiple file upload
func (h *UploadHandler) UploadDocuments(c *gin.Context) {
    form, err := c.MultipartForm()
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "multipart form required"})
        return
    }

    files := form.File["documents"]
    if len(files) == 0 {
        c.JSON(http.StatusBadRequest, gin.H{"error": "at least one document required"})
        return
    }

    saved := make([]string, 0, len(files))
    for _, file := range files {
        // Sanitize filename: strip directory components to prevent path traversal
        safeName := filepath.Base(file.Filename)
        if safeName == "." || safeName == "/" {
            c.JSON(http.StatusBadRequest, gin.H{"error": "invalid filename"})
            return
        }
        dst := filepath.Join("uploads/docs", safeName)
        if err := c.SaveUploadedFile(file, dst); err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to save file"})
            return
        }
        saved = append(saved, dst)
    }

    c.JSON(http.StatusOK, gin.H{"saved": saved, "count": len(saved)})
}
```

Route registration:

```go
r.MaxMultipartMemory = 8 << 20

protected := r.Group("/api/v1")
protected.Use(middleware.Auth(tokenCfg, logger))
{
    protected.POST("/users/:id/avatar",    uploadHandler.UploadAvatar)
    protected.POST("/users/:id/documents", uploadHandler.UploadDocuments)
}
```
