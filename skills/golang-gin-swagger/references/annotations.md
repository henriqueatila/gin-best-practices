# annotations.md — Complete Swagger Annotation Reference

Full reference for swaggo/swag annotations. Covers all `@Param` types, response patterns, file uploads, enums from constants, model renaming, grouped responses, and multiple auth schemes.

## Table of Contents

1. [Annotation Order](#1-annotation-order)
2. [General API Info](#2-general-api-info)
3. [Param Patterns](#3-param-patterns)
4. [Response Patterns](#4-response-patterns)
5. [CRUD Handler Annotation Examples](#5-crud-handler-annotation-examples)
6. [Auth Endpoint Annotations](#6-auth-endpoint-annotations)
7. [Security Definitions](#7-security-definitions)
8. [Model Tags](#8-model-tags)
9. [Enums from Constants](#9-enums-from-constants)
10. [File Uploads](#10-file-uploads)
11. [Response Headers](#11-response-headers)
12. [Model Renaming](#12-model-renaming)
13. [Deprecating Endpoints](#13-deprecating-endpoints)
14. [Tag Metadata](#14-tag-metadata)
15. [Custom Extensions](#15-custom-extensions)

---

## 1. Annotation Order

Conventional order (not enforced, but expected by `swag fmt`):

```go
// FunctionName godoc        ← standard Go doc comment (REQUIRED)
//
// @Summary      Short title
// @Description  Longer description
// @ID           unique-operation-id
// @Tags         tag1,tag2
// @Accept       json
// @Produce      json
// @Security     BearerAuth
// @Param        ...
// @Success      ...
// @Failure      ...
// @Header       ...
// @Router       /path [method]
```

Always start with a Go doc comment (`// FunctionName godoc`) followed by an empty `//` line before `@Summary`. Without the doc comment, `swag fmt` rejects the annotations.

---

## 2. General API Info

Place once in `cmd/api/main.go`, directly before `main()`:

```go
// @title           My API
// @version         1.0
// @description     Production REST API

// @termsOfService  https://example.com/terms

// @contact.name    API Support
// @contact.email   support@example.com
// @contact.url     https://example.com/support

// @license.name    MIT
// @license.url     https://opensource.org/licenses/MIT

// @host            localhost:8080
// @BasePath        /api/v1
// @schemes         http https

// @externalDocs.description  Full API documentation
// @externalDocs.url          https://docs.example.com

// Tag metadata (shows description in Swagger UI sidebar)
// @tag.name         users
// @tag.description  Operations on user accounts

// Custom extensions (vendor-specific metadata)
// @x-custom-key     {"env": "production"}

// Security definitions (see Section 5)
// @securityDefinitions.apikey  BearerAuth
// @in                          header
// @name                        Authorization
// @description                 Enter: Bearer {token}
func main() { ... }
```

Only the first annotation block is used. Do not scatter general annotations across files.

---

## 3. Param Patterns

Syntax: `@Param <name> <in> <type> <required> "<description>" [attributes]`

### Path Parameters

```go
// @Param  id    path  string  true  "User ID (UUID)"
// @Param  slug  path  string  true  "URL-friendly slug"
```

**Critical:** Use `{id}` in `@Router` (OpenAPI style), not `:id` (Gin style).

### Query Parameters

```go
// @Param  page    query  int     false  "Page number"     default(1) minimum(1)
// @Param  limit   query  int     false  "Items per page"  default(20) minimum(1) maximum(100)
// @Param  role    query  string  false  "Filter by role"  Enums(admin, user, guest)
// @Param  q       query  string  false  "Search query"    minLength(1) maxLength(200)
// @Param  sort    query  string  false  "Sort field"      default(created_at)
// @Param  order   query  string  false  "Sort order"      Enums(asc, desc) default(desc)
// @Param  active  query  bool    false  "Active only"     default(true)
```

### Header Parameters

```go
// @Param  X-Request-ID   header  string  false  "Request tracing ID"
// @Param  Accept-Language  header  string  false  "Preferred language"  default(en)
```

### Body Parameters

```go
// JSON body — reference a named struct
// @Param  request  body  domain.CreateUserRequest  true  "Request body"

// Inline primitive (rare — prefer structs)
// @Param  data  body  string  true  "Raw text payload"
```

Only one `body` param per endpoint. The struct must be importable by swag (use `--parseInternal` for `internal/` packages).

### FormData Parameters

```go
// @Param  name   formData  string  true   "User name"
// @Param  email  formData  string  true   "Email address"
// @Param  age    formData  int     false  "Age"
```

---

## 4. Response Patterns

Syntax: `@Success <code> {<type>} <model> "<description>"`

### Single Object

```go
// @Success  200  {object}  domain.User             "OK"
// @Success  201  {object}  domain.User             "Created"
```

### Array

```go
// @Success  200  {array}  domain.User  "List of users"
```

### Paginated Response

Define a wrapper struct for pagination:

```go
type PaginatedResponse struct {
    Data       []domain.User `json:"data"`
    Page       int           `json:"page"        example:"1"`
    Limit      int           `json:"limit"       example:"20"`
    TotalCount int           `json:"total_count"  example:"150"`
    TotalPages int           `json:"total_pages"  example:"8"`
}

// @Success  200  {object}  PaginatedResponse  "Paginated list"
```

### No Content

```go
// @Success  204  "No content"
```

### Primitive Response

```go
// @Success  200  {string}   string  "Plain text response"
// @Success  200  {integer}  int     "Count"
// @Success  200  {boolean}  bool    "Status"
```

### Failure Responses

Document all possible error codes:

```go
// @Failure  400  {object}  domain.ErrorResponse  "Bad request"
// @Failure  401  {object}  domain.ErrorResponse  "Unauthorized"
// @Failure  403  {object}  domain.ErrorResponse  "Forbidden"
// @Failure  404  {object}  domain.ErrorResponse  "Not found"
// @Failure  409  {object}  domain.ErrorResponse  "Conflict"
// @Failure  422  {object}  domain.ErrorResponse  "Validation failed"
// @Failure  429  {object}  domain.ErrorResponse  "Rate limit exceeded"
// @Failure  500  {object}  domain.ErrorResponse  "Internal server error"
```

### Grouped Failure Codes

When multiple codes share the same response type:

```go
// @Failure  400,401,404,500  {object}  domain.ErrorResponse
```

---

## 5. CRUD Handler Annotation Examples

Complete annotations for all standard CRUD operations. The SKILL.md covers Create, GetByID, and List. This section adds Update, Patch, Delete, and auth endpoints.

### Update (PUT — full replacement)

```go
// Update godoc
//
// @Summary      Update user
// @Description  Replace all user fields
// @Tags         users
// @Accept       json
// @Produce      json
// @Security     BearerAuth
// @Param        id     path  string                   true  "User ID" format(uuid)
// @Param        input  body  domain.UpdateUserRequest  true  "Fields to update"
// @Success      200    {object}  domain.UserResponse
// @Failure      400    {object}  domain.ErrorResponse
// @Failure      401    {object}  domain.ErrorResponse
// @Failure      404    {object}  domain.ErrorResponse
// @Failure      500    {object}  domain.ErrorResponse
// @Router       /users/{id} [put]
func (h *UserHandler) Update(c *gin.Context) {}
```

### Patch (PATCH — partial update)

```go
// Patch godoc
//
// @Summary      Partially update user
// @Description  Update only the provided fields; omitted fields are unchanged
// @Tags         users
// @Accept       json
// @Produce      json
// @Security     BearerAuth
// @Param        id     path  string                  true  "User ID" format(uuid)
// @Param        input  body  domain.PatchUserRequest  true  "Fields to patch"
// @Success      200    {object}  domain.UserResponse
// @Failure      400    {object}  domain.ErrorResponse
// @Failure      401    {object}  domain.ErrorResponse
// @Failure      404    {object}  domain.ErrorResponse
// @Failure      500    {object}  domain.ErrorResponse
// @Router       /users/{id} [patch]
func (h *UserHandler) Patch(c *gin.Context) {}
```

### Delete

```go
// Delete godoc
//
// @Summary      Delete user
// @Description  Permanently remove a user account
// @Tags         users
// @Security     BearerAuth
// @Param        id  path  string  true  "User ID" format(uuid)
// @Success      204
// @Failure      401    {object}  domain.ErrorResponse
// @Failure      403    {object}  domain.ErrorResponse
// @Failure      404    {object}  domain.ErrorResponse
// @Failure      500    {object}  domain.ErrorResponse
// @Router       /users/{id} [delete]
func (h *UserHandler) Delete(c *gin.Context) {}
```

---

## 6. Auth Endpoint Annotations

Annotate login, register, and token refresh endpoints. These do not require `@Security` because they produce tokens (not consume them).

### Register

```go
// Register godoc
//
// @Summary      Register a new account
// @Description  Create a user account and return access + refresh tokens
// @Tags         auth
// @Accept       json
// @Produce      json
// @Param        request  body      domain.RegisterRequest  true  "Registration payload"
// @Success      201      {object}  domain.TokenResponse
// @Failure      400      {object}  domain.ErrorResponse
// @Failure      409      {object}  domain.ErrorResponse   "Email already registered"
// @Failure      500      {object}  domain.ErrorResponse
// @Router       /auth/register [post]
func (h *AuthHandler) Register(c *gin.Context) {}
```

### Login

```go
// Login godoc
//
// @Summary      Log in
// @Description  Authenticate with email and password; returns access + refresh tokens
// @Tags         auth
// @Accept       json
// @Produce      json
// @Param        request  body      domain.LoginRequest  true  "Login credentials"
// @Success      200      {object}  domain.TokenResponse
// @Failure      400      {object}  domain.ErrorResponse
// @Failure      401      {object}  domain.ErrorResponse  "Invalid credentials"
// @Failure      500      {object}  domain.ErrorResponse
// @Router       /auth/login [post]
func (h *AuthHandler) Login(c *gin.Context) {}
```

### Refresh Token

```go
// RefreshToken godoc
//
// @Summary      Refresh access token
// @Description  Exchange a valid refresh token for a new access token
// @Tags         auth
// @Accept       json
// @Produce      json
// @Param        request  body      domain.RefreshRequest  true  "Refresh token payload"
// @Success      200      {object}  domain.TokenResponse
// @Failure      400      {object}  domain.ErrorResponse
// @Failure      401      {object}  domain.ErrorResponse  "Refresh token expired or invalid"
// @Failure      500      {object}  domain.ErrorResponse
// @Router       /auth/refresh [post]
func (h *AuthHandler) RefreshToken(c *gin.Context) {}
```

**Supporting DTOs for auth annotations:**

```go
// domain/auth.go

type RegisterRequest struct {
    Name     string `json:"name"     example:"Jane Doe"        binding:"required,min=2,max=100"`
    Email    string `json:"email"    example:"jane@example.com" binding:"required,email"`
    Password string `json:"password" example:"s3cur3P@ss!"      binding:"required,min=8"`
}

type LoginRequest struct {
    Email    string `json:"email"    example:"jane@example.com" binding:"required,email"`
    Password string `json:"password" example:"s3cur3P@ss!"      binding:"required"`
}

type RefreshRequest struct {
    RefreshToken string `json:"refresh_token" example:"eyJhbGci..." binding:"required"`
}

type TokenResponse struct {
    AccessToken  string `json:"access_token"  example:"eyJhbGci..."`
    RefreshToken string `json:"refresh_token" example:"eyJhbGci..."`
    ExpiresIn    int    `json:"expires_in"    example:"3600"`
}
```

---

## 7. Security Definitions

### API Key (Bearer JWT)

```go
// In main.go:
// @securityDefinitions.apikey  BearerAuth
// @in                          header
// @name                        Authorization
// @description                 Enter: Bearer {token}

// On each protected handler:
// @Security  BearerAuth
```

### API Key (Custom Header)

```go
// @securityDefinitions.apikey  ApiKeyAuth
// @in                          header
// @name                        X-API-Key
// @description                 API key for service access
```

### Basic Auth

```go
// @securityDefinitions.basic  BasicAuth
```

### OAuth2

```go
// @securityDefinitions.oauth2.implicit       OAuth2Implicit
// @authorizationUrl                          https://auth.example.com/authorize
// @scope.read                                Read access
// @scope.write                               Write access

// @securityDefinitions.oauth2.accessCode     OAuth2AccessCode
// @tokenUrl                                  https://auth.example.com/token
// @authorizationUrl                          https://auth.example.com/authorize
```

### Multiple Schemes on One Endpoint

```go
// Both required (AND logic):
// @Security  BearerAuth
// @Security  ApiKeyAuth

// Either accepted (OR logic) — use double pipe:
// @Security  BearerAuth || ApiKeyAuth
```

---

## 8. Model Tags

Complete list of struct tags recognized by swag:

```go
type Example struct {
    // Value constraints
    Name   string  `json:"name"   example:"Jane"     minLength:"2" maxLength:"100"`
    Age    int     `json:"age"    example:"30"        minimum:"0"   maximum:"150"`
    Score  float64 `json:"score"  example:"9.5"       minimum:"0"   maximum:"10"`

    // Format hints
    ID        string    `json:"id"         example:"550e8400-..."  format:"uuid"`
    Email     string    `json:"email"      example:"j@example.com" format:"email"`
    Website   string    `json:"website"    example:"https://..."   format:"uri"`
    CreatedAt time.Time `json:"created_at" example:"2024-01-15T10:30:00Z" format:"date-time"`
    Birthday  string    `json:"birthday"   example:"1990-05-15"    format:"date"`

    // Enums
    Role   string `json:"role"   example:"admin"  enums:"admin,user,guest"`
    Status string `json:"status" example:"active" enums:"active,inactive,banned"`

    // Defaults
    Page  int `json:"page"  default:"1"`
    Limit int `json:"limit" default:"20"`

    // Type overrides (for types swag can't infer)
    UpdatedAt  CustomTime    `json:"updated_at"  swaggertype:"string"  format:"date-time"`
    ExternalID sql.NullInt64 `json:"external_id" swaggertype:"integer"`
    Tags       []big.Float   `json:"tags"        swaggertype:"array,number"`
    Metadata   interface{}   `json:"metadata"    swaggertype:"object"`

    // Exclusions
    PasswordHash string `json:"-"               swaggerignore:"true"`
    InternalFlag bool   `json:"internal_flag"   swaggerignore:"true"`

    // Read-only / Write-only
    ID2 string `json:"id2" readOnly:"true"`   // shown in response only
    Pwd string `json:"pwd" writeOnly:"true"`  // shown in request only (hidden in response)

    // Required
    Required string `json:"required" binding:"required"` // swag infers required from binding tag
}
```

---

## 9. Enums from Constants

Swag auto-detects Go `const` blocks and generates enum values:

```go
type OrderStatus string

const (
    OrderPending    OrderStatus = "pending"
    OrderProcessing OrderStatus = "processing"
    OrderShipped    OrderStatus = "shipped"
    OrderDelivered  OrderStatus = "delivered"
    OrderCancelled  OrderStatus = "cancelled"
)

type Order struct {
    ID     string      `json:"id"`
    Status OrderStatus `json:"status"` // swagger generates enum automatically
}
```

This works when `OrderStatus` is a named type with `const` values of that type in the same package.

---

## 10. File Uploads

### Single File

```go
// UploadAvatar godoc
//
// @Summary      Upload user avatar
// @Tags         users
// @Accept       multipart/form-data
// @Produce      json
// @Security     BearerAuth
// @Param        id    path      string  true  "User ID"
// @Param        file  formData  file    true  "Avatar image"
// @Success      200   {object}  domain.UploadResponse
// @Failure      400   {object}  domain.ErrorResponse
// @Router       /users/{id}/avatar [post]
func (h *UserHandler) UploadAvatar(c *gin.Context) { ... }
```

### Multiple Files

```go
// @Param  files  formData  file  true  "Upload files"  collection(multi)
```

### File + Form Fields

```go
// @Accept  multipart/form-data
// @Param   name   formData  string  true  "Document name"
// @Param   file   formData  file    true  "Document file"
```

---

## 11. Response Headers

Document headers returned with responses:

```go
// @Success  200  {object}  domain.User
// @Header   200  {string}  X-Request-ID   "Unique request identifier"
// @Header   200  {integer} X-RateLimit-Remaining  "Remaining requests"
// @Header   200  {string}  X-RateLimit-Reset      "Reset timestamp"
```

---

## 12. Model Renaming

Override the model name in Swagger output (useful for disambiguating types with the same name across packages):

```go
// UserResponse is the public representation of a user.
//
// @name UserResponse
type User struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}
```

In generated docs, this appears as `UserResponse` instead of `domain.User`.

---

## 13. Deprecating Endpoints

Mark an endpoint as deprecated in docs:

```go
// GetUserV1 godoc
//
// @Summary      Get user (deprecated)
// @Description  Use GET /api/v2/users/{id} instead
// @Deprecated
// @Tags         users
// @Router       /users/{id} [get]
func (h *UserHandler) GetUserV1(c *gin.Context) { ... }
```

The endpoint appears with a strikethrough in Swagger UI.

---

## 14. Tag Metadata

Add descriptions to tags in the Swagger UI sidebar. Place in the general API info block:

```go
// @tag.name         users
// @tag.description  Operations on user accounts

// @tag.name         auth
// @tag.description  Authentication and token management

// @tag.name         admin
// @tag.description  Administrative operations (requires admin role)
```

Tags without metadata still appear in Swagger UI, but without descriptions.

---

## 15. Custom Extensions

Use `@x-` prefix for vendor-specific or custom metadata. Values are JSON:

### General API Extensions

```go
// @x-logo  {"url": "https://example.com/logo.png", "altText": "API Logo"}
```

### Operation Extensions

```go
// ListUsers godoc
//
// @Summary      List users
// @x-codeSamples  [{"lang": "curl", "source": "curl -H 'Authorization: Bearer {token}' https://api.example.com/users"}]
// @Router       /users [get]
func (h *UserHandler) List(c *gin.Context) { ... }
```

### Field Extensions

```go
type User struct {
    Role string `json:"role" x-order:"1"`
}
```

Custom extensions appear in the raw `swagger.json` output and are consumed by tools that understand them (e.g., Redoc uses `x-logo`, `x-codeSamples`).
