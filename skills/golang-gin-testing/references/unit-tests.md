# Unit Tests — Handlers, Services, and Middleware

This file covers: handler testing with `httptest`, testing authenticated routes with a mock JWT, service testing with manual mocks, middleware isolation, table-driven subtests, `t.Helper`/`t.Cleanup`/`t.Parallel`, and test fixtures.

All examples use the same `User` domain model and `AppError` pattern from **golang-gin-api**.

## Table of Contents

1. [Handler Testing with httptest](#handler-testing-with-httptest)
2. [Testing JSON and Error Responses](#testing-json-and-error-responses)
3. [Testing Authenticated Routes](#testing-authenticated-routes)
4. [Service Testing with Mocks](#service-testing-with-mocks)
5. [Mock Generation Patterns](#mock-generation-patterns)
6. [Table-Driven Tests with Subtests](#table-driven-tests-with-subtests)
7. [Test Fixtures and Factories](#test-fixtures-and-factories)
8. [t.Helper / t.Cleanup / t.Parallel](#thelper--tcleanup--tparallel)
9. [Testing Middleware in Isolation](#testing-middleware-in-isolation)

---

## Handler Testing with httptest

**Why:** Handlers translate HTTP ↔ domain. Test the HTTP contract (status codes, JSON shape, headers), not business logic. Business logic belongs in service tests.

**Pattern:** `httptest.NewRecorder()` + `router.ServeHTTP(w, req)`

```go
// internal/handler/user_handler_test.go
package handler_test

import (
    "context"
    "log/slog"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"
    "time"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
    "myapp/internal/handler"
    "myapp/internal/service"
)

func init() {
    gin.SetMode(gin.TestMode)
}

// setupRouter builds a test router with a real handler + mock service.
// Keep route registration here — it documents the contract being tested.
func setupUserRouter(svc service.UserService) *gin.Engine {
    r := gin.New() // gin.New(), not gin.Default() — no Logger noise in test output
    h := handler.NewUserHandler(svc, slog.Default())
    r.POST("/users", h.Create)
    r.GET("/users/:id", h.GetByID)
    r.PUT("/users/:id", h.Update)
    r.DELETE("/users/:id", h.Delete)
    return r
}

func TestUserHandler_GetByID(t *testing.T) {
    now := time.Now().UTC().Truncate(time.Second)
    svc := &mockUserService{
        getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
            if id == "user-123" {
                return &domain.User{
                    ID:        "user-123",
                    Name:      "Alice",
                    Email:     "alice@example.com",
                    Role:      "user",
                    CreatedAt: now,
                    UpdatedAt: now,
                }, nil
            }
            return nil, domain.ErrNotFound
        },
    }

    router := setupUserRouter(svc)

    req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("want 200, got %d; body: %s", w.Code, w.Body)
    }
    if ct := w.Header().Get("Content-Type"); !strings.Contains(ct, "application/json") {
        t.Errorf("want Content-Type application/json, got %q", ct)
    }
}
```

---

## Testing JSON and Error Responses

Verify the exact JSON shape returned — both success and error paths.

```go
// internal/handler/user_handler_json_test.go
package handler_test

import (
    "context"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "myapp/internal/domain"
)

func TestUserHandler_Create_ReturnsUserJSON(t *testing.T) {
    svc := &mockUserService{
        createFn: func(_ context.Context, req domain.CreateUserRequest) (*domain.User, error) {
            return &domain.User{
                ID:    "new-id",
                Name:  req.Name,
                Email: req.Email,
                Role:  "user",
            }, nil
        },
    }
    router := setupUserRouter(svc)

    body := `{"name":"Bob","email":"bob@example.com","password":"secret123"}`
    req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusCreated {
        t.Fatalf("want 201, got %d; body: %s", w.Code, w.Body)
    }

    var resp domain.User
    if err := json.Unmarshal(w.Body.Bytes(), &resp); err != nil {
        t.Fatalf("failed to parse response JSON: %v\nbody: %s", err, w.Body)
    }
    if resp.ID != "new-id" {
        t.Errorf("want ID 'new-id', got %q", resp.ID)
    }
    if resp.Email != "bob@example.com" {
        t.Errorf("want email 'bob@example.com', got %q", resp.Email)
    }
    // Password must never be returned
    if strings.Contains(w.Body.String(), "secret123") {
        t.Error("response body must not contain plain-text password")
    }
}

func TestUserHandler_GetByID_ErrorResponse(t *testing.T) {
    svc := &mockUserService{
        getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
            return nil, domain.ErrNotFound
        },
    }
    router := setupUserRouter(svc)

    req := httptest.NewRequest(http.MethodGet, "/users/ghost", nil)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusNotFound {
        t.Fatalf("want 404, got %d", w.Code)
    }

    var errResp map[string]string
    if err := json.Unmarshal(w.Body.Bytes(), &errResp); err != nil {
        t.Fatalf("failed to parse error JSON: %v", err)
    }
    if errResp["error"] == "" {
        t.Error("error response must contain 'error' field")
    }
}
```

---

## Testing Authenticated Routes

Inject a real JWT token generated with the test secret, then apply the `Auth` middleware in the test router.

```go
// internal/handler/protected_handler_test.go
package handler_test

import (
    "context"
    "log/slog"
    "net/http"
    "net/http/httptest"
    "testing"
    "time"

    "github.com/gin-gonic/gin"
    "myapp/internal/auth"
    "myapp/internal/domain"
    "myapp/internal/handler"
    "myapp/internal/service"
    "myapp/pkg/middleware"
)

// testTokenConfig returns a deterministic TokenConfig for tests.
func testTokenConfig() auth.TokenConfig {
    return auth.TokenConfig{
        AccessSecret:  []byte("test-access-secret-32-bytes-long!!"),
        RefreshSecret: []byte("test-refresh-secret-32-bytes-long!"),
        AccessTTL:     15 * time.Minute,
        RefreshTTL:    7 * 24 * time.Hour,
    }
}

// generateTestToken creates a valid signed JWT for test assertions.
func generateTestToken(t *testing.T, userID, email, role string) string {
    t.Helper()
    cfg := testTokenConfig()
    token, err := auth.GenerateAccessToken(cfg, userID, email, role)
    if err != nil {
        t.Fatalf("failed to generate test token: %v", err)
    }
    return token
}

func setupProtectedRouter(svc service.UserService) *gin.Engine {
    r := gin.New()
    cfg := testTokenConfig()
    protected := r.Group("")
    protected.Use(middleware.Auth(cfg, slog.Default()))
    {
        h := handler.NewUserHandler(svc, slog.Default())
        protected.GET("/users/:id", h.GetByID)
    }
    return r
}

func TestProtectedRoute_WithValidToken(t *testing.T) {
    svc := &mockUserService{
        getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
            return &domain.User{ID: id, Name: "Alice", Email: "alice@example.com", Role: "user"}, nil
        },
    }
    router := setupProtectedRouter(svc)
    token := generateTestToken(t, "user-123", "alice@example.com", "user")

    req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusOK {
        t.Errorf("want 200, got %d; body: %s", w.Code, w.Body)
    }
}

func TestProtectedRoute_MissingToken(t *testing.T) {
    router := setupProtectedRouter(&mockUserService{})

    req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
    // No Authorization header
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusUnauthorized {
        t.Errorf("want 401, got %d", w.Code)
    }
}

func TestProtectedRoute_ExpiredToken(t *testing.T) {
    cfg := auth.TokenConfig{
        AccessSecret: []byte("test-access-secret-32-bytes-long!!"),
        AccessTTL:    -1 * time.Minute, // already expired
    }
    token, err := auth.GenerateAccessToken(cfg, "u1", "a@b.com", "user")
    if err != nil {
        t.Fatal(err)
    }

    router := setupProtectedRouter(&mockUserService{})
    req := httptest.NewRequest(http.MethodGet, "/users/u1", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusUnauthorized {
        t.Errorf("want 401 for expired token, got %d", w.Code)
    }
}
```

---

## Service Testing with Mocks

Test business logic — password hashing, conflict detection, error wrapping — independent of HTTP or DB.

```go
// internal/service/user_service_test.go
package service_test

import (
    "context"
    "errors"
    "log/slog"
    "testing"

    "myapp/internal/domain"
    "myapp/internal/service"
)

type mockUserRepository struct {
    createFn     func(ctx context.Context, user *domain.User) error
    getByEmailFn func(ctx context.Context, email string) (*domain.User, error)
    getByIDFn    func(ctx context.Context, id string) (*domain.User, error)
    listFn       func(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error)
    updateFn     func(ctx context.Context, user *domain.User) error
    deleteFn     func(ctx context.Context, id string) error
}

func (m *mockUserRepository) Create(ctx context.Context, user *domain.User) error {
    if m.createFn != nil {
        return m.createFn(ctx, user)
    }
    return nil
}
func (m *mockUserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    if m.getByEmailFn != nil {
        return m.getByEmailFn(ctx, email)
    }
    return nil, domain.ErrNotFound
}
func (m *mockUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    if m.getByIDFn != nil {
        return m.getByIDFn(ctx, id)
    }
    return nil, domain.ErrNotFound
}
func (m *mockUserRepository) List(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error) {
    if m.listFn != nil {
        return m.listFn(ctx, opts)
    }
    return nil, 0, nil
}
func (m *mockUserRepository) Update(ctx context.Context, user *domain.User) error {
    if m.updateFn != nil {
        return m.updateFn(ctx, user)
    }
    return nil
}
func (m *mockUserRepository) Delete(ctx context.Context, id string) error {
    if m.deleteFn != nil {
        return m.deleteFn(ctx, id)
    }
    return nil
}

func TestUserService_Create_HashesPassword(t *testing.T) {
    var savedUser *domain.User
    repo := &mockUserRepository{
        getByEmailFn: func(_ context.Context, _ string) (*domain.User, error) {
            return nil, domain.ErrNotFound // email not taken
        },
        createFn: func(_ context.Context, user *domain.User) error {
            savedUser = user
            return nil
        },
    }

    svc := service.NewUserService(repo, slog.Default())
    _, err := svc.Create(context.Background(), domain.CreateUserRequest{
        Name:     "Alice",
        Email:    "alice@example.com",
        Password: "plaintext",
    })
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if savedUser == nil {
        t.Fatal("expected user to be saved")
    }
    // Password must be stored as bcrypt hash, not plaintext
    if savedUser.PasswordHash == "plaintext" {
        t.Error("password must not be stored as plaintext")
    }
    if savedUser.PasswordHash == "" {
        t.Error("expected PasswordHash to be set")
    }
}

func TestUserService_Create_ConflictOnDuplicateEmail(t *testing.T) {
    repo := &mockUserRepository{
        getByEmailFn: func(_ context.Context, email string) (*domain.User, error) {
            return &domain.User{Email: email}, nil // email already taken
        },
    }

    svc := service.NewUserService(repo, slog.Default())
    _, err := svc.Create(context.Background(), domain.CreateUserRequest{
        Name:     "Alice",
        Email:    "taken@example.com",
        Password: "secret123",
    })

    var appErr *domain.AppError
    if !errors.As(err, &appErr) || appErr.Code != 409 {
        t.Errorf("expected ErrConflict (409), got %v", err)
    }
}
```

---

## Mock Generation Patterns

### Manual mocks (recommended for small interfaces)

Define mock structs with function fields (shown above). Simple, explicit, no extra dependencies.

### gomock (recommended for large interfaces)

```bash
go install go.uber.org/mock/mockgen@latest

# Generate mock for UserRepository interface
mockgen -source=internal/domain/user.go -destination=internal/testutil/mocks/mock_user_repository.go -package=mocks
```

```go
// Using generated mock
import (
    "context"
    "log/slog"
    "testing"

    "go.uber.org/mock/gomock"
    "myapp/internal/domain"
    "myapp/internal/service"
    "myapp/internal/testutil/mocks"
)

func TestWithGoMock(t *testing.T) {
    ctrl := gomock.NewController(t)
    repo := mocks.NewMockUserRepository(ctrl)
    repo.EXPECT().
        GetByEmail(gomock.Any(), "alice@example.com").
        Return(nil, domain.ErrNotFound)
    repo.EXPECT().
        Create(gomock.Any(), gomock.Any()).
        Return(nil)

    svc := service.NewUserService(repo, slog.Default())
    _, err := svc.Create(context.Background(), domain.CreateUserRequest{
        Name:     "Alice",
        Email:    "alice@example.com",
        Password: "secret123",
    })
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
}
```

**Trade-off:** Manual mocks are zero-dependency and easier to debug. gomock catches unexpected calls automatically, which helps on large interfaces.

---

## Table-Driven Tests with Subtests

Structure: define a `tests` slice, loop with `t.Run`, and optionally `t.Parallel()` for speed.

```go
func TestUserService_GetByID(t *testing.T) {
    existingUser := &domain.User{ID: "u1", Name: "Alice", Email: "alice@example.com", Role: "user"}

    tests := []struct {
        name      string
        id        string
        repoUser  *domain.User
        repoErr   error
        wantUser  *domain.User
        wantErr   error
    }{
        {
            name:     "found",
            id:       "u1",
            repoUser: existingUser,
            wantUser: existingUser,
        },
        {
            name:    "not found",
            id:      "missing",
            repoErr: domain.ErrNotFound,
            wantErr: domain.ErrNotFound,
        },
        {
            name:    "repository error",
            id:      "any",
            repoErr: errors.New("db connection lost"),
            wantErr: domain.ErrInternal,
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()

            repo := &mockUserRepository{
                getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
                    return tc.repoUser, tc.repoErr
                },
            }
            svc := service.NewUserService(repo, slog.Default())
            got, err := svc.GetByID(context.Background(), tc.id)

            if tc.wantErr != nil {
                var appErr *domain.AppError
                if !errors.As(err, &appErr) {
                    t.Errorf("want AppError wrapping %v, got %T: %v", tc.wantErr, err, err)
                }
                return
            }
            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
            if got.ID != tc.wantUser.ID {
                t.Errorf("want user ID %q, got %q", tc.wantUser.ID, got.ID)
            }
        })
    }
}
```

---

## Test Fixtures and Factories

Factories build valid domain objects in one line. Keeps tests focused on what varies, not boilerplate.

```go
// internal/testutil/fixtures.go
package testutil

import (
    "time"

    "myapp/internal/domain"
)

// UserFixture returns a valid User. Override fields in the caller as needed.
func UserFixture(overrides ...func(*domain.User)) *domain.User {
    u := &domain.User{
        ID:        "fixture-user-id",
        Name:      "Test User",
        Email:     "test@example.com",
        Role:      "user",
        CreatedAt: time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC),
        UpdatedAt: time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC),
    }
    for _, fn := range overrides {
        fn(u)
    }
    return u
}

// CreateUserRequestFixture returns a valid CreateUserRequest.
func CreateUserRequestFixture(overrides ...func(*domain.CreateUserRequest)) domain.CreateUserRequest {
    req := domain.CreateUserRequest{
        Name:     "Test User",
        Email:    "test@example.com",
        Password: "secret1234",
        Role:     "user",
    }
    for _, fn := range overrides {
        fn(&req)
    }
    return req
}
```

Usage — only specify what differs from the fixture:

```go
adminUser := testutil.UserFixture(func(u *domain.User) {
    u.Role = "admin"
    u.Email = "admin@example.com"
})

noRoleReq := testutil.CreateUserRequestFixture(func(r *domain.CreateUserRequest) {
    r.Role = "" // omitempty — valid
})
```

---

## t.Helper / t.Cleanup / t.Parallel

```go
// t.Helper — marks the function as a helper, so failures show the caller's line, not the helper's
func assertStatus(t *testing.T, w *httptest.ResponseRecorder, want int) {
    t.Helper() // <-- ALWAYS add to assertion helpers
    if w.Code != want {
        t.Errorf("want status %d, got %d; body: %s", want, w.Code, w.Body)
    }
}

// t.Cleanup — deferred cleanup that runs even if the test panics
func createTempFile(t *testing.T) string {
    t.Helper()
    f, err := os.CreateTemp("", "test-*")
    if err != nil {
        t.Fatalf("createTempFile: %v", err)
    }
    t.Cleanup(func() { os.Remove(f.Name()) }) // always cleaned up
    return f.Name()
}

// t.Parallel — safe for stateless tests; do NOT use with shared mutable state
func TestHandlerConcurrency(t *testing.T) {
    t.Parallel() // this test can run concurrently with other parallel tests
    // ...
}
```

**When NOT to use `t.Parallel()`:** when tests share a database, write to files, or mutate package-level state.

---

## Testing Middleware in Isolation

Test middleware logic (JWT validation, rate limiting, RBAC) independently of any handler.

```go
// pkg/middleware/auth_test.go
package middleware_test

import (
    "log/slog"
    "net/http"
    "net/http/httptest"
    "testing"
    "time"

    "github.com/gin-gonic/gin"
    "myapp/internal/auth"
    "myapp/pkg/middleware"
)

func init() { gin.SetMode(gin.TestMode) }

// sentinel handler confirms the middleware allowed the request through
var sentinelHandler = func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"reached": true})
}

func setupAuthMiddlewareRouter(cfg auth.TokenConfig) *gin.Engine {
    r := gin.New()
    r.Use(middleware.Auth(cfg, slog.Default()))
    r.GET("/protected", sentinelHandler)
    return r
}

func TestAuthMiddleware_RejectsNoHeader(t *testing.T) {
    cfg := auth.TokenConfig{
        AccessSecret: []byte("test-secret-32-bytes-exactly-!!!"),
        AccessTTL:    15 * time.Minute,
    }
    router := setupAuthMiddlewareRouter(cfg)

    req := httptest.NewRequest(http.MethodGet, "/protected", nil)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusUnauthorized {
        t.Errorf("want 401, got %d", w.Code)
    }
}

func TestAuthMiddleware_RejectsMalformedHeader(t *testing.T) {
    cfg := auth.TokenConfig{AccessSecret: []byte("test-secret-32-bytes-exactly-!!!")}
    router := setupAuthMiddlewareRouter(cfg)

    cases := []string{"token-without-bearer", "Basic abc123", "Bearer", ""}
    for _, h := range cases {
        t.Run(h, func(t *testing.T) {
            req := httptest.NewRequest(http.MethodGet, "/protected", nil)
            if h != "" {
                req.Header.Set("Authorization", h)
            }
            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)
            if w.Code != http.StatusUnauthorized {
                t.Errorf("header=%q: want 401, got %d", h, w.Code)
            }
        })
    }
}

func TestAuthMiddleware_InjectsClaimsOnSuccess(t *testing.T) {
    cfg := auth.TokenConfig{
        AccessSecret: []byte("test-secret-32-bytes-exactly-!!!"),
        AccessTTL:    15 * time.Minute,
    }

    // Capture injected claims in a custom handler
    var capturedUserID string
    r := gin.New()
    r.Use(middleware.Auth(cfg, slog.Default()))
    r.GET("/me", func(c *gin.Context) {
        capturedUserID = c.GetString(middleware.UserIDKey)
        c.JSON(http.StatusOK, gin.H{"ok": true})
    })

    token, err := auth.GenerateAccessToken(cfg, "user-xyz", "u@example.com", "user")
    if err != nil {
        t.Fatal(err)
    }

    req := httptest.NewRequest(http.MethodGet, "/me", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("want 200, got %d; body: %s", w.Code, w.Body)
    }
    if capturedUserID != "user-xyz" {
        t.Errorf("want user_id 'user-xyz' injected by middleware, got %q", capturedUserID)
    }
}
```
