# Integration Tests — Real Database with testcontainers

This file covers: test database setup with testcontainers-go, `TestMain` for DB lifecycle, cleanup between tests, repository integration tests, API integration tests with a running router, build tags, and fixture loading.

**When to use:** Write integration tests when you need to verify actual SQL behavior — migrations, GORM/sqlx queries, constraint enforcement, transaction rollbacks. Unit tests with mocked repositories cannot catch these.

## Table of Contents

1. [Dependencies](#dependencies)
2. [Build Tags](#build-tags)
3. [TestMain — DB Lifecycle](#testmain--db-lifecycle)
4. [Test Database Helper](#test-database-helper)
5. [Cleanup Between Tests](#cleanup-between-tests)
6. [Repository Integration Tests](#repository-integration-tests)
7. [API Integration Tests](#api-integration-tests)
8. [Testing Migrations](#testing-migrations)
9. [Fixture Loading](#fixture-loading)

---

## Dependencies

```bash
go get github.com/testcontainers/testcontainers-go
go get github.com/testcontainers/testcontainers-go/modules/postgres
go get gorm.io/driver/postgres
go get github.com/golang-migrate/migrate/v4
```

Requires Docker running on the host (or in CI). Each test run starts a real PostgreSQL container.

---

## Build Tags

Isolate integration tests from the unit test run using build tags. This keeps `go test ./...` fast.

```go
//go:build integration
// +build integration
```

Place this at the **top of every integration test file**, before the `package` declaration.

```bash
# Run unit tests only (fast, default CI step)
go test -v -race -cover -tags='!integration' ./...

# Run integration tests (requires Docker)
go test -v -race -tags=integration ./internal/repository/...

# Run all tests
go test -v -race -tags=integration ./...
```

---

## TestMain — DB Lifecycle

`TestMain` runs once per package, starts the container, runs all tests, then cleans up. This amortizes container startup cost across all tests in the package.

```go
//go:build integration

// internal/repository/integration_test.go
package repository_test

import (
    "context"
    "fmt"
    "log/slog"
    "os"
    "testing"
    "time"

    "github.com/testcontainers/testcontainers-go"
    tcpostgres "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    gormpostgres "gorm.io/driver/postgres"
    "gorm.io/gorm"

    "myapp/internal/repository"
)

var (
    testDB        *gorm.DB
    testContainer testcontainers.Container
)

func TestMain(m *testing.M) {
    ctx := context.Background()

    // Start PostgreSQL container
    pgContainer, err := tcpostgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        tcpostgres.WithDatabase("testdb"),
        tcpostgres.WithUsername("testuser"),
        tcpostgres.WithPassword("testpass"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(30*time.Second),
        ),
    )
    if err != nil {
        slog.Error("failed to start postgres container", "error", err)
        os.Exit(1)
    }
    testContainer = pgContainer

    // Build DSN from container
    host, err := pgContainer.Host(ctx)
    if err != nil {
        slog.Error("failed to get container host", "error", err)
        cleanupContainer(ctx)
        os.Exit(1)
    }
    port, err := pgContainer.MappedPort(ctx, "5432")
    if err != nil {
        slog.Error("failed to get mapped port", "error", err)
        cleanupContainer(ctx)
        os.Exit(1)
    }
    dsn := fmt.Sprintf("host=%s port=%s user=testuser password=testpass dbname=testdb sslmode=disable",
        host, port.Port())

    // Connect with GORM
    db, err := gorm.Open(gormpostgres.Open(dsn), &gorm.Config{})
    if err != nil {
        slog.Error("failed to connect to test db", "error", err)
        cleanupContainer(ctx)
        os.Exit(1)
    }
    testDB = db

    // Run migrations
    if err := runTestMigrations(db); err != nil {
        slog.Error("migrations failed", "error", err)
        cleanupContainer(ctx)
        os.Exit(1)
    }

    // Run all tests in this package
    code := m.Run()

    // Cleanup
    cleanupContainer(ctx)
    os.Exit(code)
}

func cleanupContainer(ctx context.Context) {
    if testContainer != nil {
        if err := testContainer.Terminate(ctx); err != nil {
            slog.Error("failed to terminate container", "error", err)
        }
    }
}

func runTestMigrations(db *gorm.DB) error {
    // Auto-migrate models for integration tests
    return db.AutoMigrate(&repository.UserModel{})
}
```

---

## Test Database Helper

Encapsulate container setup in a reusable helper so multiple packages can share the same pattern.

```go
//go:build integration

// internal/testutil/testdb/postgres_container.go
package testdb

import (
    "context"
    "fmt"
    "testing"
    "time"

    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    pgdriver "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

// NewPostgres starts a PostgreSQL container and returns a connected *gorm.DB.
// The container is terminated when the test completes via t.Cleanup.
func NewPostgres(t *testing.T, models ...any) *gorm.DB {
    t.Helper()
    ctx := context.Background()

    pgContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("testuser"),
        postgres.WithPassword("testpass"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(30*time.Second),
        ),
    )
    if err != nil {
        t.Fatalf("testdb.NewPostgres: start container: %v", err)
    }
    t.Cleanup(func() {
        if err := pgContainer.Terminate(ctx); err != nil {
            t.Logf("testdb.NewPostgres: terminate container: %v", err)
        }
    })

    host, err := pgContainer.Host(ctx)
    if err != nil {
        t.Fatalf("testdb.NewPostgres: get host: %v", err)
    }
    port, err := pgContainer.MappedPort(ctx, "5432")
    if err != nil {
        t.Fatalf("testdb.NewPostgres: get mapped port: %v", err)
    }
    dsn := fmt.Sprintf("host=%s port=%s user=testuser password=testpass dbname=testdb sslmode=disable",
        host, port.Port())

    db, err := gorm.Open(pgdriver.Open(dsn), &gorm.Config{})
    if err != nil {
        t.Fatalf("testdb.NewPostgres: gorm.Open: %v", err)
    }

    if len(models) > 0 {
        if err := db.AutoMigrate(models...); err != nil {
            t.Fatalf("testdb.NewPostgres: AutoMigrate: %v", err)
        }
    }

    return db
}

// Truncate clears all rows from the given tables.
// Call between tests to avoid state leaking across test cases.
//
// SAFETY: table names must be compile-time constants from your codebase.
// Never pass user-supplied input here — PostgreSQL does not support
// parameterized DDL statements, so string concatenation is unavoidable.
// The caller is responsible for ensuring only known, hardcoded table names
// are passed (e.g., Truncate(t, db, "users", "orders")).
func Truncate(t *testing.T, db *gorm.DB, tables ...string) {
    t.Helper()
    for _, table := range tables {
        if err := db.Exec("TRUNCATE TABLE " + table + " RESTART IDENTITY CASCADE").Error; err != nil {
            t.Fatalf("testdb.Truncate(%q): %v", table, err)
        }
    }
}
```

---

## Cleanup Between Tests

Each test must start with a clean database state. Two patterns:

### Pattern 1: Truncate in t.Cleanup (recommended)

```go
//go:build integration

func TestRepository_Create(t *testing.T) {
    // Truncate users table after this test completes (runs even on failure)
    t.Cleanup(func() {
        testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    })

    repo := repository.NewUserRepository(testDB)
    user := &domain.User{
        ID:    "u1",
        Name:  "Alice",
        Email: "alice@example.com",
        Role:  "user",
    }

    err := repo.Create(context.Background(), user)
    if err != nil {
        t.Fatalf("Create: %v", err)
    }
}
```

### Pattern 2: Transaction rollback per test

Wrap each test in a transaction that is always rolled back. Fast, but doesn't test transaction commit behavior.

```go
//go:build integration

func withTx(t *testing.T, db *gorm.DB, fn func(tx *gorm.DB)) {
    t.Helper()
    tx := db.Begin()
    if tx.Error != nil {
        t.Fatalf("begin tx: %v", tx.Error)
    }
    t.Cleanup(func() { tx.Rollback() })
    fn(tx)
}

func TestRepository_GetByID_InTx(t *testing.T) {
    withTx(t, testDB, func(tx *gorm.DB) {
        repo := repository.NewUserRepository(tx)
        // All writes are rolled back after the test
        _ = repo.Create(context.Background(), &domain.User{
            ID: "tmp", Name: "Temp", Email: "tmp@example.com", Role: "user",
        })
        got, err := repo.GetByID(context.Background(), "tmp")
        if err != nil {
            t.Fatalf("GetByID: %v", err)
        }
        if got.Name != "Temp" {
            t.Errorf("want 'Temp', got %q", got.Name)
        }
    })
}
```

---

## Repository Integration Tests

Test actual SQL behavior: constraints, not-found semantics, list filtering, and pagination.

```go
//go:build integration

// internal/repository/user_repository_integration_test.go
package repository_test

import (
    "context"
    "errors"
    "testing"

    "myapp/internal/domain"
    "myapp/internal/repository"
)

func TestUserRepository_Create_AndGetByID(t *testing.T) {
    t.Cleanup(func() {
        testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    })

    repo := repository.NewUserRepository(testDB)
    ctx := context.Background()

    user := &domain.User{
        ID:    "int-test-id",
        Name:  "Alice",
        Email: "alice@example.com",
        Role:  "user",
    }

    if err := repo.Create(ctx, user); err != nil {
        t.Fatalf("Create: %v", err)
    }

    got, err := repo.GetByID(ctx, user.ID)
    if err != nil {
        t.Fatalf("GetByID: %v", err)
    }
    if got.Email != user.Email {
        t.Errorf("want email %q, got %q", user.Email, got.Email)
    }
}

func TestUserRepository_GetByID_NotFound(t *testing.T) {
    repo := repository.NewUserRepository(testDB)

    _, err := repo.GetByID(context.Background(), "does-not-exist")
    if !errors.Is(err, domain.ErrNotFound) {
        t.Errorf("want ErrNotFound, got %v", err)
    }
}

func TestUserRepository_Create_DuplicateEmail_ReturnsConflict(t *testing.T) {
    t.Cleanup(func() {
        testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    })

    repo := repository.NewUserRepository(testDB)
    ctx := context.Background()

    first := &domain.User{ID: "u1", Name: "Alice", Email: "alice@example.com", Role: "user"}
    if err := repo.Create(ctx, first); err != nil {
        t.Fatalf("Create first: %v", err)
    }

    duplicate := &domain.User{ID: "u2", Name: "Alice2", Email: "alice@example.com", Role: "user"}
    err := repo.Create(ctx, duplicate)
    if !errors.Is(err, domain.ErrConflict) {
        t.Errorf("want ErrConflict on duplicate email, got %v", err)
    }
}

func TestUserRepository_List_FiltersByRole(t *testing.T) {
    t.Cleanup(func() {
        testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    })

    repo := repository.NewUserRepository(testDB)
    ctx := context.Background()

    _ = repo.Create(ctx, &domain.User{ID: "u1", Name: "Admin", Email: "admin@example.com", Role: "admin"})
    _ = repo.Create(ctx, &domain.User{ID: "u2", Name: "User1", Email: "user1@example.com", Role: "user"})
    _ = repo.Create(ctx, &domain.User{ID: "u3", Name: "User2", Email: "user2@example.com", Role: "user"})

    users, total, err := repo.List(ctx, domain.ListOptions{Page: 1, Limit: 10, Role: "user"})
    if err != nil {
        t.Fatalf("List: %v", err)
    }
    if total != 2 {
        t.Errorf("want 2 users with role 'user', got %d", total)
    }
    if len(users) != 2 {
        t.Errorf("want 2 returned users, got %d", len(users))
    }
}

func TestUserRepository_Delete_SoftDelete(t *testing.T) {
    t.Cleanup(func() {
        testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    })

    repo := repository.NewUserRepository(testDB)
    ctx := context.Background()

    user := &domain.User{ID: "del-me", Name: "Alice", Email: "alice@example.com", Role: "user"}
    if err := repo.Create(ctx, user); err != nil {
        t.Fatalf("Create: %v", err)
    }

    if err := repo.Delete(ctx, user.ID); err != nil {
        t.Fatalf("Delete: %v", err)
    }

    _, err := repo.GetByID(ctx, user.ID)
    if !errors.Is(err, domain.ErrNotFound) {
        t.Errorf("want ErrNotFound after delete, got %v", err)
    }
}
```

---

## API Integration Tests

Test the full stack — real router with real database — by calling `router.ServeHTTP`. No network socket needed.

```go
//go:build integration

// internal/handler/user_handler_integration_test.go
package handler_test

import (
    "encoding/json"
    "log/slog"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
    "myapp/internal/handler"
    "myapp/internal/repository"
    "myapp/internal/service"
    "myapp/internal/testutil/testdb"
)

func init() { gin.SetMode(gin.TestMode) }

func buildIntegrationRouter(t *testing.T) (*gin.Engine, func()) {
    t.Helper()
    db := testdb.NewPostgres(t, &repository.UserModel{})
    repo := repository.NewUserRepository(db)
    svc := service.NewUserService(repo, slog.Default())
    h := handler.NewUserHandler(svc, slog.Default())

    r := gin.New()
    r.POST("/users", h.Create)
    r.GET("/users/:id", h.GetByID)

    cleanup := func() {
        db.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    }
    return r, cleanup
}

func TestUserAPI_CreateAndRetrieve(t *testing.T) {
    router, cleanup := buildIntegrationRouter(t)
    t.Cleanup(cleanup)

    // Step 1: Create user
    body := `{"name":"Alice","email":"alice@example.com","password":"secret1234"}`
    req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusCreated {
        t.Fatalf("Create: want 201, got %d; body: %s", w.Code, w.Body)
    }

    var created domain.User
    if err := json.Unmarshal(w.Body.Bytes(), &created); err != nil {
        t.Fatalf("parse create response: %v", err)
    }
    if created.ID == "" {
        t.Fatal("expected non-empty ID in response")
    }

    // Step 2: Retrieve the same user
    req2 := httptest.NewRequest(http.MethodGet, "/users/"+created.ID, nil)
    w2 := httptest.NewRecorder()
    router.ServeHTTP(w2, req2)

    if w2.Code != http.StatusOK {
        t.Fatalf("GetByID: want 200, got %d; body: %s", w2.Code, w2.Body)
    }

    var fetched domain.User
    if err := json.Unmarshal(w2.Body.Bytes(), &fetched); err != nil {
        t.Fatalf("parse get response: %v", err)
    }
    if fetched.Email != "alice@example.com" {
        t.Errorf("want email 'alice@example.com', got %q", fetched.Email)
    }
}
```

---

## Testing Migrations

Verify migrations apply cleanly and the schema matches the expected structure.

```go
//go:build integration

// internal/repository/migrations_integration_test.go
package repository_test

import (
    "context"
    "fmt"
    "testing"
    "time"

    "github.com/golang-migrate/migrate/v4"
    migratepg "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
    "github.com/testcontainers/testcontainers-go"
    tcpostgres "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func TestMigrations_UpAndDown(t *testing.T) {
    ctx := context.Background()

    pgc, err := tcpostgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        tcpostgres.WithDatabase("migrate_test"),
        tcpostgres.WithUsername("u"),
        tcpostgres.WithPassword("p"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(20*time.Second),
        ),
    )
    if err != nil {
        t.Fatalf("start container: %v", err)
    }
    t.Cleanup(func() { pgc.Terminate(ctx) })

    host, err := pgc.Host(ctx)
    if err != nil {
        t.Fatalf("get container host: %v", err)
    }
    port, err := pgc.MappedPort(ctx, "5432")
    if err != nil {
        t.Fatalf("get mapped port: %v", err)
    }
    dsn := fmt.Sprintf("host=%s port=%s user=u password=p dbname=migrate_test sslmode=disable",
        host, port.Port())

    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        t.Fatalf("gorm.Open: %v", err)
    }
    sqlDB, _ := db.DB()

    driver, err := migratepg.WithInstance(sqlDB, &migratepg.Config{})
    if err != nil {
        t.Fatalf("migrate driver: %v", err)
    }

    m, err := migrate.NewWithDatabaseInstance("file://../../migrations", "postgres", driver)
    if err != nil {
        t.Fatalf("migrate.New: %v", err)
    }

    // Apply all migrations
    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        t.Fatalf("migrate Up: %v", err)
    }

    // Verify expected tables exist
    var tableName string
    row := sqlDB.QueryRow("SELECT table_name FROM information_schema.tables WHERE table_schema='public' AND table_name='users'")
    if err := row.Scan(&tableName); err != nil {
        t.Fatalf("users table not found after migration: %v", err)
    }

    // Roll back all migrations
    if err := m.Down(); err != nil && err != migrate.ErrNoChange {
        t.Fatalf("migrate Down: %v", err)
    }
}
```

---

## Fixture Loading

Load SQL fixture files to populate a known database state before tests.

```go
//go:build integration

// internal/testutil/testdb/fixtures.go
package testdb

import (
    "os"
    "testing"

    "gorm.io/gorm"
)

// LoadFixture executes SQL from a fixture file against the given db.
// Fixture files live in internal/testutil/fixtures/*.sql
func LoadFixture(t *testing.T, db *gorm.DB, path string) {
    t.Helper()

    sql, err := os.ReadFile(path)
    if err != nil {
        t.Fatalf("LoadFixture: read %q: %v", path, err)
    }

    if err := db.Exec(string(sql)).Error; err != nil {
        t.Fatalf("LoadFixture: exec %q: %v", path, err)
    }
}
```

Example fixture file `internal/testutil/fixtures/users.sql`:

```sql
INSERT INTO users (id, name, email, role, created_at, updated_at)
VALUES
    ('seed-user-1', 'Alice',  'alice@example.com', 'user',  NOW(), NOW()),
    ('seed-user-2', 'Bob',    'bob@example.com',   'user',  NOW(), NOW()),
    ('seed-admin',  'Admin',  'admin@example.com', 'admin', NOW(), NOW());
```

Usage in a test:

```go
//go:build integration

func TestUserRepository_List_WithFixtures(t *testing.T) {
    t.Cleanup(func() {
        testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    })

    testdb.LoadFixture(t, testDB, "../../testutil/fixtures/users.sql")

    repo := repository.NewUserRepository(testDB)
    users, total, err := repo.List(context.Background(), domain.ListOptions{Page: 1, Limit: 10})
    if err != nil {
        t.Fatalf("List: %v", err)
    }
    if total != 3 {
        t.Errorf("want 3 users from fixture, got %d", total)
    }
    _ = users
}
```
