# API Design Patterns for Go Gin APIs

Deep-dive reference for API versioning, pagination, filtering, bulk operations, deprecation, and backwards compatibility. For quick rules, see the SKILL.md.

## Table of Contents

- [Versioning Strategies](#versioning-strategies)
- [Pagination](#pagination)
- [Filtering and Sorting](#filtering-and-sorting)
- [Bulk Operations](#bulk-operations)
- [Partial Updates](#partial-updates)
- [API Evolution and Deprecation](#api-evolution-and-deprecation)
- [Backwards Compatibility Rules](#backwards-compatibility-rules)
- [Error Response Contract](#error-response-contract)
- [HATEOAS — When You Need It](#hateoas)
- [API Documentation](#api-documentation)

---

## Versioning Strategies

### URL Path Versioning (Recommended)

Simplest, most visible, easiest to test and debug.

```go
func registerRoutes(r *gin.Engine, v1 V1Handlers, v2 V2Handlers) {
    apiV1 := r.Group("/api/v1")
    {
        apiV1.GET("/users", v1.ListUsers)
        apiV1.GET("/users/:id", v1.GetUser)
        apiV1.POST("/users", v1.CreateUser)
    }

    apiV2 := r.Group("/api/v2")
    {
        apiV2.GET("/users", v2.ListUsers)    // new pagination format
        apiV2.GET("/users/:id", v2.GetUser)  // expanded response
        apiV2.POST("/users", v2.CreateUser)  // same as v1
    }
}
```

**When to bump the version:**
- Removing a field from a response
- Changing a field type (string → int)
- Changing the meaning of a field
- Changing pagination format
- Removing an endpoint

**When NOT to bump:**
- Adding a new field to a response
- Adding a new endpoint
- Adding a new query parameter
- Fixing a bug in behavior

### Header Versioning (Not Recommended for Most Cases)

```
Accept: application/vnd.myapp.v2+json
```

Harder to test (can't paste URL in browser), harder to cache, harder to route. Use only if you have a strong reason (e.g., many API versions with minimal differences).

### Query Parameter Versioning (Avoid)

```
GET /api/users?version=2
```

Pollutes query strings, caching nightmares. Don't use.

---

## Pagination

### Cursor-Based (Recommended for Large Sets)

Best for: feeds, timelines, any list that changes frequently. No "page drift" problem.

```go
// Request: GET /api/v1/orders?cursor=eyJpZCI6MTAwfQ&limit=20
type CursorQuery struct {
    Cursor string `form:"cursor"`
    Limit  int    `form:"limit" binding:"min=1,max=100"`
}

// Response
type CursorPage[T any] struct {
    Data       []T    `json:"data"`
    NextCursor string `json:"next_cursor,omitempty"` // empty = last page
    HasMore    bool   `json:"has_more"`
}

func (h *OrderHandler) List(c *gin.Context) {
    q := CursorQuery{Limit: 20} // default
    if err := c.ShouldBindQuery(&q); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    orders, nextCursor, err := h.svc.ListOrders(c.Request.Context(), q.Cursor, q.Limit)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, CursorPage[domain.Order]{
        Data:       orders,
        NextCursor: nextCursor,
        HasMore:    nextCursor != "",
    })
}
```

**Cursor implementation in repository:**

```go
func (r *OrderRepo) List(ctx context.Context, cursor string, limit int) ([]domain.Order, string, error) {
    var lastID int64
    if cursor != "" {
        decoded, err := base64.StdEncoding.DecodeString(cursor)
        if err != nil {
            return nil, "", fmt.Errorf("invalid cursor: %w", err)
        }
        lastID, _ = strconv.ParseInt(string(decoded), 10, 64)
    }

    query := `SELECT id, user_id, status, total, created_at
              FROM orders
              WHERE ($1 = 0 OR id < $1)
              ORDER BY id DESC
              LIMIT $2`

    var orders []domain.Order
    if err := r.db.SelectContext(ctx, &orders, query, lastID, limit+1); err != nil {
        return nil, "", fmt.Errorf("list orders: %w", err)
    }

    var nextCursor string
    if len(orders) > limit {
        lastOrder := orders[limit]
        nextCursor = base64.StdEncoding.EncodeToString([]byte(strconv.FormatInt(lastOrder.ID, 10)))
        orders = orders[:limit]
    }

    return orders, nextCursor, nil
}
```

### Offset-Based (OK for Small Sets)

Best for: admin dashboards, internal tools, data < 10K rows.

```go
// Request: GET /api/v1/users?page=1&limit=20
type OffsetQuery struct {
    Page  int `form:"page"  binding:"min=1"`
    Limit int `form:"limit" binding:"min=1,max=100"`
}

type OffsetPage[T any] struct {
    Data       []T `json:"data"`
    Page       int `json:"page"`
    Limit      int `json:"limit"`
    TotalCount int `json:"total_count"`
    TotalPages int `json:"total_pages"`
}
```

**Warning:** Offset pagination degrades with large datasets (`OFFSET 100000` scans 100K rows). Switch to cursor-based when data grows.

---

## Filtering and Sorting

### Query Parameter Filtering

```go
// GET /api/v1/orders?status=confirmed&min_total=1000&created_after=2026-01-01
type OrderFilter struct {
    Status       string `form:"status"        binding:"omitempty,oneof=draft confirmed shipped delivered"`
    MinTotal     *int64 `form:"min_total"     binding:"omitempty,min=0"`
    MaxTotal     *int64 `form:"max_total"     binding:"omitempty,min=0"`
    CreatedAfter string `form:"created_after" binding:"omitempty,datetime=2006-01-02"`
}
```

**Rules:**
- Use flat query params for simple filters
- Use pointer types for optional filters (`*int64` → nil means "not filtered")
- Validate enum values with `oneof`
- Date formats as `YYYY-MM-DD` or RFC3339

### Sorting

```go
// GET /api/v1/orders?sort=created_at&order=desc
type SortQuery struct {
    Sort  string `form:"sort"  binding:"omitempty,oneof=created_at total status"`
    Order string `form:"order" binding:"omitempty,oneof=asc desc"`
}

func (q SortQuery) OrderClause() string {
    col := q.Sort
    if col == "" {
        col = "created_at"
    }
    dir := q.Order
    if dir == "" {
        dir = "desc"
    }
    // Safe — values are validated by binding tags
    return col + " " + dir
}
```

**Security:** Always allowlist sortable columns. Never interpolate raw user input into ORDER BY.

---

## Bulk Operations

### Bulk Create

```go
// POST /api/v1/users/bulk
type BulkCreateRequest struct {
    Users []domain.CreateUserRequest `json:"users" binding:"required,min=1,max=100,dive"`
}

type BulkCreateResponse struct {
    Created []domain.User  `json:"created"`
    Errors  []BulkError    `json:"errors,omitempty"`
}

type BulkError struct {
    Index   int    `json:"index"`
    Message string `json:"message"`
}
```

**Rules:**
- Cap batch size (100 is a good default)
- Use `dive` tag to validate each item in the array
- Return partial results — some may succeed, some may fail
- Use database transactions for all-or-nothing semantics when needed

### Bulk Delete

```go
// DELETE /api/v1/users/bulk
type BulkDeleteRequest struct {
    IDs []string `json:"ids" binding:"required,min=1,max=100"`
}
```

---

## Partial Updates

### PATCH with JSON Merge

```go
// PATCH /api/v1/users/:id
// Body: {"name": "New Name"} — only updates name, keeps other fields

type UpdateUserRequest struct {
    Name  *string `json:"name"  binding:"omitempty,min=2,max=100"`
    Email *string `json:"email" binding:"omitempty,email"`
    Role  *string `json:"role"  binding:"omitempty,oneof=admin user"`
}

func (h *UserHandler) Update(c *gin.Context) {
    id := c.Param("id")

    var req UpdateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.svc.Update(c.Request.Context(), id, req)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, user)
}
```

**Key:** Use pointer types (`*string`) to distinguish "not sent" (nil) from "sent as empty" (`""`).

---

## API Evolution and Deprecation

### Deprecation Headers

```go
func DeprecatedEndpoint(sunset string) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Deprecation", "true")
        c.Header("Sunset", sunset) // RFC 7231 date: "Sat, 01 Jun 2026 00:00:00 GMT"
        c.Header("Link", `</api/v2/users>; rel="successor-version"`)
        c.Next()
    }
}

// Usage
apiV1.GET("/users", DeprecatedEndpoint("Sat, 01 Jun 2026 00:00:00 GMT"), v1.ListUsers)
```

### Migration Guide Pattern

When introducing a breaking change:
1. Ship v2 endpoint alongside v1
2. Add deprecation headers to v1
3. Log v1 usage to track migration progress
4. Communicate sunset date (minimum 3 months for external APIs)
5. Remove v1 after sunset date

---

## Backwards Compatibility Rules

### Safe Changes (No Version Bump)

| Change | Why Safe |
|---|---|
| Add new field to response | Clients should ignore unknown fields |
| Add new endpoint | Doesn't affect existing endpoints |
| Add new optional query parameter | Existing calls work without it |
| Add new enum value | Clients should handle unknown values gracefully |
| Widen validation (accept more input) | Previously valid input is still valid |

### Breaking Changes (Require Version Bump)

| Change | Why Breaking |
|---|---|
| Remove field from response | Clients may depend on it |
| Rename field | Same as remove + add |
| Change field type | Deserialization breaks |
| Narrow validation (reject previously valid input) | Existing clients may break |
| Change error codes/format | Client error handling breaks |
| Remove endpoint | Clients can't call it |
| Change URL structure | Existing URLs break |

### The Robustness Principle

> Be conservative in what you send, be liberal in what you accept.

- Accept unknown fields in requests (don't error on extra JSON keys)
- Ignore unknown fields in responses (clients should not break on new fields)
- Use `json:",omitempty"` for optional response fields

---

## Error Response Contract

Consistent across all endpoints and versions:

```go
// Standard error response
type ErrorResponse struct {
    Error   string            `json:"error"`              // human-readable
    Code    string            `json:"code,omitempty"`     // machine-readable: "USER_NOT_FOUND"
    Details map[string]string `json:"details,omitempty"`  // field-level errors
}

// Validation error response
// POST /api/v1/users with invalid body
// 422 Unprocessable Entity
{
    "error": "validation failed",
    "code": "VALIDATION_ERROR",
    "details": {
        "email": "invalid email format",
        "name": "must be at least 2 characters"
    }
}

// Not found
// GET /api/v1/users/999
// 404 Not Found
{
    "error": "user not found",
    "code": "USER_NOT_FOUND"
}
```

---

## HATEOAS

**You probably don't need this.** HATEOAS adds complexity for minimal benefit in most REST APIs. Use it only if your API is truly generic and consumed by clients that discover capabilities dynamically.

For most Gin APIs: document your endpoints in OpenAPI/Swagger and call it a day.

---

## API Documentation

### OpenAPI/Swagger with swaggo

```go
// Install: go install github.com/swaggo/swag/cmd/swag@latest

// main.go annotations
// @title           My API
// @version         1.0
// @description     A production API built with Gin
// @host            localhost:8080
// @BasePath        /api/v1
// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization

// Handler annotations
// @Summary      Create user
// @Description  Create a new user account
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        body body domain.CreateUserRequest true "User data"
// @Success      201  {object} domain.User
// @Failure      400  {object} ErrorResponse
// @Failure      409  {object} ErrorResponse
// @Router       /users [post]
func (h *UserHandler) Create(c *gin.Context) { ... }
```

Generate: `swag init -g cmd/api/main.go`

Serve: use `github.com/swaggo/gin-swagger` middleware at `/swagger/*`.

---

## Quick Decision: Which Pagination?

```
START: How many total records?
  ├── < 10K → Offset pagination is fine
  └── > 10K → Is the data frequently changing (inserts/deletes)?
      ├── No → Offset still OK (but monitor performance)
      └── Yes → Cursor-based pagination
```
