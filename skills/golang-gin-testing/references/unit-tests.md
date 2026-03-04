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
10. [Benchmark Tests](#benchmark-tests)
11. [Fuzz Tests](#fuzz-tests)
12. [Golden File / Snapshot Testing](#golden-file--snapshot-testing)
13. [Test Organization](#test-organization)
14. [Testify as an Alternative](#testify-as-an-alternative)

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

---

## Benchmark Tests

Use `Benchmark*` functions to measure performance of hot paths — JSON serialization, handler throughput, service logic. Run with `go test -bench=. -benchmem ./...`.

```go
// internal/handler/user_handler_bench_test.go
package handler_test

import (
    "context"
    "net/http"
    "net/http/httptest"
    "testing"

    "myapp/internal/domain"
)

// BenchmarkUserHandler_GetByID measures handler throughput including
// JSON marshaling and routing overhead.
func BenchmarkUserHandler_GetByID(b *testing.B) {
    svc := &mockUserService{
        getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
            return &domain.User{
                ID:    id,
                Name:  "Alice",
                Email: "alice@example.com",
                Role:  "user",
            }, nil
        },
    }
    router := setupUserRouter(svc)

    b.ReportAllocs() // report allocations per operation
    b.ResetTimer()

    for b.Loop() {
        req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
        w := httptest.NewRecorder()
        router.ServeHTTP(w, req)
        if w.Code != http.StatusOK {
            b.Fatalf("unexpected status %d", w.Code)
        }
    }
}

// BenchmarkUserHandler_GetByID_Parallel runs the same benchmark
// with multiple goroutines to expose concurrency bottlenecks.
func BenchmarkUserHandler_GetByID_Parallel(b *testing.B) {
    svc := &mockUserService{
        getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
            return &domain.User{ID: id, Name: "Alice", Email: "alice@example.com", Role: "user"}, nil
        },
    }
    router := setupUserRouter(svc)

    b.ReportAllocs()
    b.ResetTimer()

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)
        }
    })
}
```

Run benchmarks:

```bash
# All benchmarks in the package
go test -bench=. -benchmem ./internal/handler/...

# Specific benchmark, 5 seconds run time
go test -bench=BenchmarkUserHandler_GetByID -benchtime=5s -benchmem ./internal/handler/...

# Compare two versions with benchstat
go test -bench=. -benchmem -count=5 ./... > before.txt
# (make changes)
go test -bench=. -benchmem -count=5 ./... > after.txt
benchstat before.txt after.txt
```

---

## Fuzz Tests

Go 1.18+ native fuzzing (`func FuzzX(f *testing.F)`) finds edge cases in input parsing that table-driven tests miss. Use for request binding, validators, and string-processing utilities.

```go
// internal/handler/user_handler_fuzz_test.go
package handler_test

import (
    "context"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "myapp/internal/domain"
)

// FuzzUserHandler_Create exercises the Create handler with arbitrary JSON bodies.
// The fuzzer discovers inputs that cause panics or unexpected status codes.
//
// Run the fuzzer: go test -fuzz=FuzzUserHandler_Create ./internal/handler/...
// Run the corpus only (fast, CI-safe): go test ./internal/handler/...
func FuzzUserHandler_Create(f *testing.F) {
    // Seed corpus — representative inputs the fuzzer mutates from
    f.Add(`{"name":"Alice","email":"alice@example.com","password":"secret123"}`)
    f.Add(`{"name":"","email":"bad-email","password":"x"}`)
    f.Add(`{}`)
    f.Add(`not-json`)
    f.Add(`{"name":null,"email":null}`)

    svc := &mockUserService{
        createFn: func(_ context.Context, req domain.CreateUserRequest) (*domain.User, error) {
            return &domain.User{ID: "fuzz-id", Name: req.Name, Email: req.Email, Role: "user"}, nil
        },
    }
    router := setupUserRouter(svc)

    f.Fuzz(func(t *testing.T, body string) {
        req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
        req.Header.Set("Content-Type", "application/json")
        w := httptest.NewRecorder()
        router.ServeHTTP(w, req)

        // The handler must NEVER panic (recovered by Gin) and must always
        // return a valid HTTP status — never a 5xx for invalid client input.
        if w.Code == http.StatusInternalServerError {
            t.Errorf("handler returned 500 for input %q; body: %s", body, w.Body)
        }
    })
}
```

**Key rules for fuzz tests:**
- Seed corpus must include valid inputs and known edge cases
- The fuzz function must not have non-deterministic behavior (no random, no time)
- Run in CI with `-fuzz=` omitted (seed corpus only) — fuzzing itself runs locally or on dedicated infrastructure
- Found crash inputs are saved to `testdata/fuzz/FuzzX/` automatically

---

## Golden File / Snapshot Testing

Golden files capture the expected output of a function as a file on disk. On subsequent runs the output is compared to the stored file. Useful for JSON responses, rendered templates, or any output too verbose to inline.

```go
// internal/handler/user_handler_golden_test.go
package handler_test

import (
    "encoding/json"
    "flag"
    "os"
    "path/filepath"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "myapp/internal/domain"
)

// Run with -update to regenerate golden files:
//   go test ./internal/handler/... -update
var update = flag.Bool("update", false, "update golden files")

func TestUserResponse_Golden(t *testing.T) {
    user := domain.User{ID: "test-id", Email: "test@example.com", Name: "Test User"}
    body, err := json.MarshalIndent(user, "", "  ")
    require.NoError(t, err)

    golden := filepath.Join("testdata", t.Name()+".golden")
    if *update {
        os.MkdirAll("testdata", 0o755)
        os.WriteFile(golden, body, 0o644)
    }
    expected, err := os.ReadFile(golden)
    require.NoError(t, err)
    assert.JSONEq(t, string(expected), string(body))
}
```

**How it works:**

1. First run: `go test ./... -update` writes `testdata/TestUserResponse_Golden.golden`
2. Subsequent runs compare current output to the stored file
3. Commit golden files to source control — diffs surface unintended response changes
4. When the response shape intentionally changes, re-run with `-update` and commit the new file

Golden files live in `testdata/` next to the test file. The `testdata/` directory is ignored by the Go build toolchain but committed to git.

---

## Test Organization

Go supports two complementary test styles in the same package directory:

### Same-package tests (white-box)

```go
// File: internal/service/user_service_test.go
package service  // same package — can access unexported identifiers
```

Use for: verifying internal state, unexported helper functions, implementation details that should not be part of the public API.

### External-package tests (black-box)

```go
// File: internal/handler/user_handler_test.go
package handler_test  // _test suffix — only exported API visible
```

Use for: handlers, middleware, repositories. Tests the public contract; prevents accidental coupling to internals.

**Rule of thumb:** handlers and services use `package X_test` (black-box). Internal utility packages may use `package X` (white-box) when testing unexported logic.

### Build tags for integration tests

```go
//go:build integration

// Place at the top of every integration test file, before the package declaration.
// Excluded from: go test ./...
// Included in:   go test -tags=integration ./...
```

Use `//go:build integration` for tests that require Docker/network. Use `//go:build e2e` for full end-to-end tests. This keeps `go test ./...` fast for everyday development.

---

## Testify as an Alternative

The standard library `testing` package is sufficient for most cases. In enterprise codebases, [testify](https://github.com/testify-community/testify) (`github.com/stretchr/testify`) is widely adopted for its readable assertions and reduced boilerplate.

```bash
go get github.com/stretchr/testify
```

**Comparison:**

```go
// Standard library
if got.Email != "alice@example.com" {
    t.Errorf("want email 'alice@example.com', got %q", got.Email)
}
if err != nil {
    t.Fatalf("unexpected error: %v", err)
}

// testify/assert — continues test on failure
assert.Equal(t, "alice@example.com", got.Email)
assert.NoError(t, err)

// testify/require — stops test immediately on failure (equivalent to t.Fatalf)
require.NoError(t, err)
require.Equal(t, "alice@example.com", got.Email)
```

**When to use each:**

| Package | Behavior | Use when |
|---|---|---|
| `assert` | non-fatal, test continues | checking multiple independent fields |
| `require` | fatal, stops immediately | precondition must hold for rest of test to make sense |

`require.NoError(t, err)` before accessing the result is the most common pattern — no point checking `got.Email` if `err != nil` crashed the response.

**Trade-off:** testify adds a dependency but significantly reduces assertion noise in large test suites. The standard library is always available with zero dependencies.
