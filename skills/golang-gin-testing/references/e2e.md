# End-to-End Tests — Critical Flow Testing

This file covers: e2e test structure, critical user flows (register → login → CRUD), testing with docker-compose, GitHub Actions CI/CD integration, test environment configuration, and cleanup/idempotency strategies.

**When to use:** Write e2e tests for flows that span multiple services or require the full wiring to be correct — auth token generation, protected route access, cascading deletes. Not every endpoint needs an e2e test; focus on critical user journeys.

## Table of Contents

1. [E2E Test Structure](#e2e-test-structure)
2. [Critical Flow: Register → Login → CRUD](#critical-flow-register--login--crud)
3. [In-Process E2E with httptest](#in-process-e2e-with-httptest)
4. [Testing with docker-compose](#testing-with-docker-compose)
5. [GitHub Actions CI/CD Integration](#github-actions-cicd-integration)
6. [Test Environment Configuration](#test-environment-configuration)
7. [Cleanup and Idempotency](#cleanup-and-idempotency)

---

## E2E Test Structure

E2e tests wire the **entire application** — router, middleware, handlers, services, repositories — against a real database. They verify that the wiring is correct and critical flows work end-to-end.

```
internal/
└── e2e/                        # e2e test package (separate from unit/integration)
    ├── e2e_test.go             # TestMain, app setup
    ├── user_flow_test.go       # register → login → CRUD flow
    └── auth_flow_test.go       # token refresh, RBAC flows
```

**Build tag:** Use `//go:build e2e` to exclude from normal test runs.

```bash
# Run e2e tests only
go test -v -tags=e2e ./internal/e2e/...

# Run all tests including e2e
go test -v -tags='integration e2e' ./...
```

---

## Critical Flow: Register → Login → CRUD

The most important e2e flow: create a user, authenticate, then perform authenticated operations. If this breaks, the entire API is unusable.

```go
//go:build e2e

// internal/e2e/user_flow_test.go
package e2e_test

import (
    "encoding/json"
    "fmt"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "myapp/internal/domain"
)

// testApp is initialized once in TestMain (see e2e_test.go).
// It holds the fully wired router and a cleanup function.
var testApp *appUnderTest

func TestUserFlow_RegisterLoginCRUD(t *testing.T) {
    t.Cleanup(func() { testApp.truncate(t, "users") })

    // Step 1: Register a new user
    registerBody := `{"name":"Alice","email":"alice@e2e.com","password":"secret1234"}`
    w := testApp.do(t, http.MethodPost, "/api/v1/users", registerBody, "")

    if w.Code != http.StatusCreated {
        t.Fatalf("register: want 201, got %d; body: %s", w.Code, w.Body)
    }
    var created domain.User
    mustUnmarshal(t, w, &created)
    if created.ID == "" {
        t.Fatal("register: expected non-empty user ID")
    }

    // Step 2: Login to get a JWT
    loginBody := `{"email":"alice@e2e.com","password":"secret1234"}`
    w = testApp.do(t, http.MethodPost, "/api/v1/auth/login", loginBody, "")

    if w.Code != http.StatusOK {
        t.Fatalf("login: want 200, got %d; body: %s", w.Code, w.Body)
    }
    var tokens struct {
        AccessToken  string `json:"access_token"`
        RefreshToken string `json:"refresh_token"`
    }
    mustUnmarshal(t, w, &tokens)
    if tokens.AccessToken == "" {
        t.Fatal("login: expected non-empty access_token")
    }

    // Step 3: Fetch own profile with the JWT
    w = testApp.do(t, http.MethodGet, "/api/v1/users/"+created.ID, "", tokens.AccessToken)
    if w.Code != http.StatusOK {
        t.Fatalf("get user: want 200, got %d; body: %s", w.Code, w.Body)
    }
    var fetched domain.User
    mustUnmarshal(t, w, &fetched)
    if fetched.Email != "alice@e2e.com" {
        t.Errorf("get user: want email 'alice@e2e.com', got %q", fetched.Email)
    }

    // Step 4: Update own profile
    updateBody := fmt.Sprintf(`{"name":"Alice Updated","email":"alice@e2e.com"}`)
    w = testApp.do(t, http.MethodPut, "/api/v1/users/"+created.ID, updateBody, tokens.AccessToken)
    if w.Code != http.StatusOK {
        t.Fatalf("update user: want 200, got %d; body: %s", w.Code, w.Body)
    }

    // Step 5: Delete own account
    w = testApp.do(t, http.MethodDelete, "/api/v1/users/"+created.ID, "", tokens.AccessToken)
    if w.Code != http.StatusNoContent {
        t.Fatalf("delete user: want 204, got %d; body: %s", w.Code, w.Body)
    }

    // Step 6: Confirm gone
    w = testApp.do(t, http.MethodGet, "/api/v1/users/"+created.ID, "", tokens.AccessToken)
    if w.Code != http.StatusNotFound {
        t.Errorf("after delete: want 404, got %d", w.Code)
    }
}

func TestUserFlow_LoginWithWrongPassword(t *testing.T) {
    t.Cleanup(func() { testApp.truncate(t, "users") })

    // Register first
    testApp.do(t, http.MethodPost, "/api/v1/users",
        `{"name":"Bob","email":"bob@e2e.com","password":"secret1234"}`, "")

    // Wrong password → 401
    w := testApp.do(t, http.MethodPost, "/api/v1/auth/login",
        `{"email":"bob@e2e.com","password":"wrongpassword"}`, "")
    if w.Code != http.StatusUnauthorized {
        t.Errorf("wrong password: want 401, got %d", w.Code)
    }
}

func TestUserFlow_ProtectedRoute_WithoutToken(t *testing.T) {
    w := testApp.do(t, http.MethodGet, "/api/v1/users/any-id", "", "")
    if w.Code != http.StatusUnauthorized {
        t.Errorf("no token: want 401, got %d", w.Code)
    }
}

func mustUnmarshal(t *testing.T, w *httptest.ResponseRecorder, dst any) {
    t.Helper()
    if err := json.Unmarshal(w.Body.Bytes(), dst); err != nil {
        t.Fatalf("mustUnmarshal: %v\nbody: %s", err, w.Body)
    }
}
```

---

## In-Process E2E with httptest

Wire the full application in-process — no network socket, no external process. Fast, deterministic, and reproducible.

```go
//go:build e2e

// internal/e2e/e2e_test.go
package e2e_test

import (
    "context"
    "log/slog"
    "net/http"
    "net/http/httptest"
    "os"
    "strings"
    "testing"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/testcontainers/testcontainers-go"
    tcpostgres "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    pgdriver "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "fmt"

    "myapp/internal/auth"
    "myapp/internal/handler"
    "myapp/internal/repository"
    "myapp/internal/service"
    "myapp/pkg/middleware"
)

// appUnderTest holds the fully wired router and test helpers.
type appUnderTest struct {
    router *gin.Engine
    db     *gorm.DB
}

// do performs an HTTP request against the in-process router.
func (a *appUnderTest) do(t *testing.T, method, path, body, token string) *httptest.ResponseRecorder {
    t.Helper()
    var r *strings.Reader
    if body != "" {
        r = strings.NewReader(body)
    } else {
        r = strings.NewReader("")
    }

    req, err := http.NewRequest(method, path, r)
    if err != nil {
        t.Fatalf("do: http.NewRequest: %v", err)
    }
    if body != "" {
        req.Header.Set("Content-Type", "application/json")
    }
    if token != "" {
        req.Header.Set("Authorization", "Bearer "+token)
    }

    w := httptest.NewRecorder()
    a.router.ServeHTTP(w, req)
    return w
}

// truncate clears rows between tests.
// SAFETY: table names must be compile-time constants (e.g., "users", "orders").
// Never pass user-supplied input — PostgreSQL DDL does not support parameterized
// table names, so string concatenation is used. Restrict callers to hardcoded names.
func (a *appUnderTest) truncate(t *testing.T, tables ...string) {
    t.Helper()
    for _, tbl := range tables {
        if err := a.db.Exec("TRUNCATE TABLE " + tbl + " RESTART IDENTITY CASCADE").Error; err != nil {
            t.Logf("truncate %q: %v", tbl, err)
        }
    }
}

// buildApp wires the full application for e2e tests.
func buildApp(db *gorm.DB) *appUnderTest {
    gin.SetMode(gin.TestMode)
    logger := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{Level: slog.LevelWarn}))

    tokenCfg := auth.TokenConfig{
        AccessSecret:  []byte("e2e-test-access-secret-32-bytes!!"),
        RefreshSecret: []byte("e2e-test-refresh-secret-32-bytes!"),
        AccessTTL:     15 * time.Minute,
        RefreshTTL:    7 * 24 * time.Hour,
    }

    userRepo    := repository.NewUserRepository(db)
    userSvc     := service.NewUserService(userRepo, logger)
    userHandler := handler.NewUserHandler(userSvc, logger)
    authHandler := handler.NewAuthHandler(userRepo, tokenCfg, logger)

    r := gin.New() // gin.New() — explicit middleware, no defaults
    r.Use(middleware.Recovery(logger))

    api := r.Group("/api/v1")

    public := api.Group("")
    {
        public.POST("/users", userHandler.Create)
        public.POST("/auth/login", authHandler.Login)
        public.POST("/auth/refresh", authHandler.Refresh)
    }

    protected := api.Group("")
    protected.Use(middleware.Auth(tokenCfg, logger))
    {
        protected.GET("/users/:id", userHandler.GetByID)
        protected.PUT("/users/:id", userHandler.Update)
        protected.DELETE("/users/:id", userHandler.Delete)
    }

    return &appUnderTest{router: r, db: db}
}

func TestMain(m *testing.M) {
    ctx := context.Background()

    pgc, err := tcpostgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        tcpostgres.WithDatabase("e2edb"),
        tcpostgres.WithUsername("e2euser"),
        tcpostgres.WithPassword("e2epass"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(30*time.Second),
        ),
    )
    if err != nil {
        slog.Error("e2e: start postgres container", "error", err)
        os.Exit(1)
    }

    host, err := pgc.Host(ctx)
    if err != nil {
        slog.Error("e2e: get container host", "error", err)
        pgc.Terminate(ctx)
        os.Exit(1)
    }
    port, err := pgc.MappedPort(ctx, "5432")
    if err != nil {
        slog.Error("e2e: get mapped port", "error", err)
        pgc.Terminate(ctx)
        os.Exit(1)
    }
    dsn := fmt.Sprintf("host=%s port=%s user=e2euser password=e2epass dbname=e2edb sslmode=disable",
        host, port.Port())

    db, err := gorm.Open(pgdriver.Open(dsn), &gorm.Config{})
    if err != nil {
        slog.Error("e2e: gorm.Open", "error", err)
        pgc.Terminate(ctx)
        os.Exit(1)
    }

    if err := db.AutoMigrate(&repository.UserModel{}); err != nil {
        slog.Error("e2e: AutoMigrate", "error", err)
        pgc.Terminate(ctx)
        os.Exit(1)
    }

    testApp = buildApp(db)

    code := m.Run()
    pgc.Terminate(ctx)
    os.Exit(code)
}
```

---

## Testing with docker-compose

For e2e tests that need external services (Redis, message queues), use a docker-compose file dedicated to testing.

```yaml
# docker-compose.test.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    ports:
      - "5433:5432"       # 5433 to avoid colliding with dev postgres on 5432
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser -d testdb"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6380:6379"       # 6380 to avoid colliding with dev redis
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

Run tests against the compose stack:

```bash
# Start infrastructure
docker compose -f docker-compose.test.yml up -d --wait

# Run e2e tests against external services
E2E_DATABASE_URL="postgres://testuser:testpass@localhost:5433/testdb?sslmode=disable" \
E2E_REDIS_URL="redis://localhost:6380" \
go test -v -tags=e2e ./internal/e2e/...

# Teardown
docker compose -f docker-compose.test.yml down -v
```

Reading the DSN from the environment in e2e setup:

```go
//go:build e2e

func dsnFromEnv(t *testing.T) string {
    t.Helper()
    dsn := os.Getenv("E2E_DATABASE_URL")
    if dsn == "" {
        // Fall back to testcontainers if env var not set
        return "" // signal to use testcontainers
    }
    return dsn
}
```

---

## GitHub Actions CI/CD Integration

Run unit, integration, and e2e tests in separate jobs. Integration and e2e jobs use the `services` block to spin up Postgres.

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  unit:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.24"
          cache: true

      - name: Run unit tests
        run: go test -v -race -cover -tags='!integration !e2e' ./...

      - name: Check coverage threshold
        run: |
          go test ./... -coverprofile=coverage.out -tags='!integration !e2e'
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage ${COVERAGE}% is below 80% threshold"
            exit 1
          fi
          echo "Coverage: ${COVERAGE}%"

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  integration:
    name: Integration Tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U testuser -d testdb"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.24"
          cache: true

      - name: Run integration tests
        env:
          TEST_DATABASE_URL: postgres://testuser:testpass@localhost:5432/testdb?sslmode=disable
        # testcontainers auto-detects Docker; for CI using services block, pass DSN via env
        run: go test -v -race -tags=integration ./internal/repository/...

  e2e:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: [unit, integration]   # only run if earlier jobs pass
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: e2edb
          POSTGRES_USER: e2euser
          POSTGRES_PASSWORD: e2epass
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U e2euser -d e2edb"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.24"
          cache: true

      - name: Run e2e tests
        env:
          E2E_DATABASE_URL: postgres://e2euser:e2epass@localhost:5432/e2edb?sslmode=disable
        run: go test -v -race -tags=e2e -timeout=120s ./internal/e2e/...
```

---

## Test Environment Configuration

Keep test configuration separate from production config. Load from environment with sensible test defaults.

```go
// internal/e2e/config_test.go
//go:build e2e

package e2e_test

import (
    "os"
    "time"

    "myapp/internal/auth"
)

// e2eConfig holds environment-driven configuration for e2e tests.
// All values have safe defaults so tests run without any env setup (uses testcontainers).
type e2eConfig struct {
    DatabaseURL string
    TokenConfig auth.TokenConfig
    Timeout     time.Duration
}

func loadE2EConfig() e2eConfig {
    return e2eConfig{
        // Empty string → use testcontainers (resolved in TestMain)
        DatabaseURL: os.Getenv("E2E_DATABASE_URL"),
        TokenConfig: auth.TokenConfig{
            AccessSecret:  []byte(envOr("E2E_JWT_SECRET", "e2e-test-access-secret-32-bytes!!")),
            RefreshSecret: []byte(envOr("E2E_JWT_REFRESH_SECRET", "e2e-test-refresh-secret-32!!!!!")),
            AccessTTL:     15 * time.Minute,
            RefreshTTL:    7 * 24 * time.Hour,
        },
        Timeout: 30 * time.Second,
    }
}

func envOr(key, fallback string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return fallback
}
```

---

## Cleanup and Idempotency

E2e tests must clean up after themselves so they can run in any order and be re-run without manual resets.

### Rules

1. **Each test owns its data** — create what you need, clean up in `t.Cleanup`.
2. **Never depend on data from another test** — tests must be independent.
3. **Use unique identifiers** — email addresses, IDs should be unique per test run to avoid conflicts.
4. **Truncate, don't drop** — truncating preserves the schema; dropping requires re-migration.

```go
//go:build e2e

// Good: each test cleans up its own data
func TestUserFlow_CreateAndDelete(t *testing.T) {
    t.Cleanup(func() {
        testApp.truncate(t, "users")
    })
    // ... test body
}

// Good: unique email per test to avoid duplicate conflicts
func uniqueEmail(t *testing.T) string {
    t.Helper()
    // Use test name as suffix — safe for parallel tests
    safe := strings.NewReplacer("/", "-", " ", "-").Replace(t.Name())
    return fmt.Sprintf("user+%s@e2e.com", strings.ToLower(safe))
}

func TestUserFlow_ParallelSafe(t *testing.T) {
    t.Parallel()
    email := uniqueEmail(t) // e.g. "user+testuserflow_parallelsafe@e2e.com"
    t.Cleanup(func() {
        testApp.db.Exec("DELETE FROM users WHERE email = ?", email)
    })

    testApp.do(t, http.MethodPost, "/api/v1/users",
        fmt.Sprintf(`{"name":"Test","email":%q,"password":"secret1234"}`, email), "")
    // ...
}
```

### Idempotency checklist

- [ ] Test passes on first run
- [ ] Test passes on second run without manual cleanup
- [ ] Test passes when run in parallel with other tests
- [ ] `t.Cleanup` registered before any assertions (runs even if test fails)
- [ ] No hardcoded IDs or emails that conflict across runs
