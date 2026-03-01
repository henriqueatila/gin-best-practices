# sqlx Patterns Reference

This file covers sqlx-specific patterns for PostgreSQL integration in Gin APIs: connection setup, struct scanning, Get/Select/NamedExec, safe dynamic queries, transactions, sqlx.In for IN clauses, null handling, query builder patterns, connection pooling, and a complete repository implementation satisfying the same `domain.UserRepository` interface as the GORM version. Use when building the repository layer with sqlx instead of GORM.

> **Architectural recommendation:** These are mainstream Go/sqlx community patterns, not part of the Gin framework API.

## Table of Contents

1. [Connection Setup](#connection-setup)
2. [Struct Scanning with db Tags](#struct-scanning-with-db-tags)
3. [Get, Select, NamedExec](#get-select-namedexec)
4. [Safe Dynamic Queries](#safe-dynamic-queries)
5. [Transactions with sqlx.Tx](#transactions-with-sqlxtx)
6. [sqlx.In for IN Clauses](#sqlxin-for-in-clauses)
7. [Null Handling](#null-handling)
8. [Query Builder Patterns](#query-builder-patterns)
9. [Connection Pooling](#connection-pooling)
10. [Complete Repository Implementation](#complete-repository-implementation)

---

## Connection Setup

```go
// internal/repository/db_sqlx.go
package repository

import (
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq" // PostgreSQL driver — blank import registers it
)

type SqlxConfig struct {
    DSN             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
}

// NewSqlxDB opens a PostgreSQL connection via sqlx.
// Architectural recommendation — not part of the Gin API.
func NewSqlxDB(cfg SqlxConfig, logger *slog.Logger) (*sqlx.DB, error) {
    db, err := sqlx.Connect("postgres", cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("sqlx.Connect: %w", err)
    }

    db.SetMaxOpenConns(cfg.MaxOpenConns)       // e.g. 25
    db.SetMaxIdleConns(cfg.MaxIdleConns)       // e.g. 5
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime)  // e.g. 5*time.Minute

    // sqlx.Connect already pings — this confirms pool is healthy
    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("db.Ping: %w", err)
    }

    logger.Info("database connected (sqlx)")
    return db, nil
}
```

**sqlx vs GORM:** sqlx wraps `database/sql` with struct scanning and named queries. It executes raw SQL — no ORM magic, no hidden queries, full control. Choose sqlx when you need complex queries or existing DBA-crafted SQL.

---

## Struct Scanning with db Tags

Map SQL columns to struct fields using `db` tags. No GORM, no magic.

```go
// internal/repository/user_row.go
package repository

import (
    "time"

    "myapp/internal/domain"
)

// userRow is the sqlx scan target for the users table.
// Kept in the repository package — never exposed to domain or handlers.
type userRow struct {
    ID        string    `db:"id"`
    Name      string    `db:"name"`
    Email     string    `db:"email"`
    Role      string    `db:"role"`
    CreatedAt time.Time `db:"created_at"`
    UpdatedAt time.Time `db:"updated_at"`
}

func (r *userRow) toDomain() *domain.User {
    return &domain.User{
        ID:        r.ID,
        Name:      r.Name,
        Email:     r.Email,
        Role:      r.Role,
        CreatedAt: r.CreatedAt,
        UpdatedAt: r.UpdatedAt,
    }
}

// userInsertRow includes fields only used on INSERT (e.g. password_hash).
type userInsertRow struct {
    ID           string `db:"id"`
    Name         string `db:"name"`
    Email        string `db:"email"`
    PasswordHash string `db:"password_hash"`
    Role         string `db:"role"`
}
```

**Rule:** Column names in SQL must exactly match `db` tags. Mismatches produce silent zero values — always verify with a test query.

---

## Get, Select, NamedExec

sqlx adds three key methods on top of `database/sql`:

| Method | Returns | Use for |
|--------|---------|---------|
| `GetContext` | single row into struct | `SELECT ... WHERE id = $1` |
| `SelectContext` | slice of structs | `SELECT ... WHERE ...` many rows |
| `NamedExecContext` | `sql.Result` | INSERT/UPDATE with named params |

```go
// GetContext — single row; returns sql.ErrNoRows if not found
var u userRow
err := db.GetContext(ctx, &u, `
    SELECT id, name, email, role, created_at, updated_at
    FROM users
    WHERE id = $1 AND deleted_at IS NULL
`, id)

// SelectContext — multiple rows
var rows []userRow
err := db.SelectContext(ctx, &rows, `
    SELECT id, name, email, role, created_at, updated_at
    FROM users
    WHERE deleted_at IS NULL
    ORDER BY created_at DESC
    LIMIT $1 OFFSET $2
`, limit, offset)

// NamedExecContext — named parameters from struct or map
row := userInsertRow{
    ID:           newUUID(),
    Name:         req.Name,
    Email:        req.Email,
    PasswordHash: hashPassword(req.Password),
    Role:         req.Role,
}
_, err := db.NamedExecContext(ctx, `
    INSERT INTO users (id, name, email, password_hash, role)
    VALUES (:id, :name, :email, :password_hash, :role)
`, row)
```

**Critical:** Never use `fmt.Sprintf` to build SQL strings — use `$N` placeholders for positional params or `:name` for named params. This prevents SQL injection.

---

## Safe Dynamic Queries

Build dynamic WHERE clauses safely using `squirrel` or manual slice accumulation. Never string-concatenate user input.

```go
// Option A: manual accumulation — no extra dependency
func buildListQuery(opts domain.ListOptions) (string, []any) {
    conditions := []string{"deleted_at IS NULL"}
    args := []any{}
    argIdx := 1

    if opts.Role != "" {
        conditions = append(conditions, fmt.Sprintf("role = $%d", argIdx))
        args = append(args, opts.Role)
        argIdx++
    }

    where := strings.Join(conditions, " AND ")
    limit := opts.Limit
    if limit <= 0 || limit > 100 {
        limit = 20
    }
    offset := (opts.Page - 1) * limit

    query := fmt.Sprintf(`
        SELECT id, name, email, role, created_at, updated_at
        FROM users
        WHERE %s
        ORDER BY created_at DESC
        LIMIT $%d OFFSET $%d
    `, where, argIdx, argIdx+1)

    args = append(args, limit, offset)
    return query, args
}

// Option B: squirrel query builder (github.com/Masterminds/squirrel)
import sq "github.com/Masterminds/squirrel"

psql := sq.StatementBuilder.PlaceholderFormat(sq.Dollar)

qb := psql.Select("id", "name", "email", "role", "created_at", "updated_at").
    From("users").
    Where("deleted_at IS NULL")

if opts.Role != "" {
    qb = qb.Where(sq.Eq{"role": opts.Role})
}

sql, args, err := qb.OrderBy("created_at DESC").
    Limit(uint64(limit)).
    Offset(uint64(offset)).
    ToSql()
```

---

## Transactions with sqlx.Tx

```go
// internal/repository/tx_sqlx.go
package repository

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"
)

// WithTransaction runs fn inside a transaction, rolling back on error.
func WithTransaction(ctx context.Context, db *sqlx.DB, fn func(*sqlx.Tx) error) error {
    tx, err := db.BeginTxx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }

    if err := fn(tx); err != nil {
        if rbErr := tx.Rollback(); rbErr != nil {
            return fmt.Errorf("tx rollback failed: %v (original: %w)", rbErr, err)
        }
        return err
    }

    return tx.Commit()
}

// Service-layer usage — the service orchestrates the transaction:
func (s *userService) RegisterWithProfile(ctx context.Context, req domain.CreateUserRequest) error {
    return repository.WithTransaction(ctx, s.db, func(tx *sqlx.Tx) error {
        if err := s.userRepo.CreateTx(ctx, tx, &user); err != nil {
            return err
        }
        return s.profileRepo.CreateTx(ctx, tx, &profile)
    })
}

// Repository method accepting a tx — dual interface pattern
func (r *sqlxUserRepository) CreateTx(ctx context.Context, tx *sqlx.Tx, user *domain.User) error {
    row := toInsertRow(user)
    _, err := tx.NamedExecContext(ctx, insertUserSQL, row)
    if err != nil {
        return mapSqlxError(err)
    }
    return nil
}
```

**Pattern choice:** Passing `*sqlx.Tx` explicitly is simpler than context-based tx propagation for sqlx. The context approach (storing tx in ctx) works but requires type assertions at every repository method.

---

## sqlx.In for IN Clauses

`database/sql` does not expand `IN (?)` for slices — sqlx.In does.

```go
import "github.com/jmoiron/sqlx"

// sqlx.In expands the slice into positional placeholders
func (r *sqlxUserRepository) GetByIDs(ctx context.Context, ids []string) ([]domain.User, error) {
    if len(ids) == 0 {
        return nil, nil
    }

    // sqlx.In rewrites "IN (?)" → "IN ($1,$2,$3,...)"
    query, args, err := sqlx.In(`
        SELECT id, name, email, role, created_at, updated_at
        FROM users
        WHERE id IN (?) AND deleted_at IS NULL
    `, ids)
    if err != nil {
        return nil, domain.ErrInternal.New(err)
    }

    // Rebind converts ? placeholders to $N for PostgreSQL
    query = r.db.Rebind(query)

    var rows []userRow
    if err := r.db.SelectContext(ctx, &rows, query, args...); err != nil {
        return nil, domain.ErrInternal.New(err)
    }

    users := make([]domain.User, len(rows))
    for i, row := range rows {
        users[i] = *row.toDomain()
    }
    return users, nil
}
```

**Warning:** Large IN clauses (1000+ IDs) can hurt performance. For bulk lookups, prefer a temporary table or `ANY($1::uuid[])` with a PostgreSQL array parameter.

---

## Null Handling

PostgreSQL NULLs map to Go's zero values unless you use `database/sql` null types.

```go
import "database/sql"

// userRowNullable handles columns that can be NULL in the database
type userRowNullable struct {
    ID        string         `db:"id"`
    Name      string         `db:"name"`
    Email     string         `db:"email"`
    Role      string         `db:"role"`
    DeletedAt sql.NullTime   `db:"deleted_at"` // NULL when not deleted
    Bio       sql.NullString `db:"bio"`         // NULL when profile incomplete
    CreatedAt time.Time      `db:"created_at"`
    UpdatedAt time.Time      `db:"updated_at"`
}

func (r *userRowNullable) toDomain() *domain.User {
    u := &domain.User{
        ID:        r.ID,
        Name:      r.Name,
        Email:     r.Email,
        Role:      r.Role,
        CreatedAt: r.CreatedAt,
        UpdatedAt: r.UpdatedAt,
    }
    if r.Bio.Valid {
        u.Bio = r.Bio.String
    }
    return u
}

// Alternative: use github.com/guregu/null for ergonomic null types
import "gopkg.in/guregu/null.v4"

type userRowNullV2 struct {
    Bio null.String `db:"bio"` // null.String.ValueOrZero() returns "" if null
}
```

---

## Query Builder Patterns

Keep SQL in constants or a dedicated file for discoverability.

```go
// internal/repository/user_queries.go
package repository

const (
    insertUserSQL = `
        INSERT INTO users (id, name, email, password_hash, role, created_at, updated_at)
        VALUES (:id, :name, :email, :password_hash, :role, NOW(), NOW())
    `

    selectUserByIDSQL = `
        SELECT id, name, email, role, created_at, updated_at
        FROM users
        WHERE id = $1 AND deleted_at IS NULL
    `

    selectUserByEmailSQL = `
        SELECT id, name, email, role, created_at, updated_at
        FROM users
        WHERE email = $1 AND deleted_at IS NULL
    `

    updateUserSQL = `
        UPDATE users
        SET name = :name, role = :role, updated_at = NOW()
        WHERE id = :id AND deleted_at IS NULL
    `

    softDeleteUserSQL = `
        UPDATE users SET deleted_at = NOW() WHERE id = $1 AND deleted_at IS NULL
    `

    countUsersByRoleSQL = `
        SELECT COUNT(*) FROM users WHERE deleted_at IS NULL
    `
)
```

**Why constants?** Grep-able, type-checkable at compile time, and easy to test in isolation with `sqlx.NamedExec` dry-run.

---

## Connection Pooling

Same recommendations as GORM (both use `database/sql` under the hood):

```go
db.SetMaxOpenConns(25)                     // max concurrent connections
db.SetMaxIdleConns(5)                      // warm idle pool
db.SetConnMaxLifetime(5 * time.Minute)     // recycle stale connections
db.SetConnMaxIdleTime(1 * time.Minute)     // close idle connections sooner
```

**Health check endpoint** — ping the database to confirm connectivity:

```go
// internal/handler/health_handler.go
func (h *HealthHandler) Check(c *gin.Context) {
    if err := h.db.PingContext(c.Request.Context()); err != nil {
        c.JSON(http.StatusServiceUnavailable, gin.H{
            "status": "degraded",
            "db":     "unreachable",
        })
        return
    }
    c.JSON(http.StatusOK, gin.H{"status": "ok"})
}
```

---

## Complete Repository Implementation

Full `UserRepository` satisfying the same `domain.UserRepository` interface as the GORM version — swap implementations in `main.go` without changing service code.

```go
// internal/repository/user_repository_sqlx.go
package repository

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
    "strings"

    "github.com/google/uuid"
    "github.com/jmoiron/sqlx"
    "myapp/internal/domain"
)

type sqlxUserRepository struct {
    db *sqlx.DB
}

// NewSqlxUserRepository returns a domain.UserRepository backed by sqlx.
// Drop-in replacement for NewUserRepository (GORM version).
func NewSqlxUserRepository(db *sqlx.DB) domain.UserRepository {
    return &sqlxUserRepository{db: db}
}

func (r *sqlxUserRepository) Create(ctx context.Context, user *domain.User) error {
    if user.ID == "" {
        user.ID = newUUID()
    }
    if user.Role == "" {
        user.Role = "user"
    }

    row := userInsertRow{
        ID:    user.ID,
        Name:  user.Name,
        Email: user.Email,
        Role:  user.Role,
    }
    _, err := r.db.NamedExecContext(ctx, insertUserSQL, row)
    if err != nil {
        return mapSqlxError(err)
    }
    return nil
}

func (r *sqlxUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    var row userRow
    if err := r.db.GetContext(ctx, &row, selectUserByIDSQL, id); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return row.toDomain(), nil
}

func (r *sqlxUserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    var row userRow
    if err := r.db.GetContext(ctx, &row, selectUserByEmailSQL, email); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return row.toDomain(), nil
}

func (r *sqlxUserRepository) List(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error) {
    // Count
    var total int64
    countQuery, countArgs := buildCountQuery(opts)
    if err := r.db.GetContext(ctx, &total, countQuery, countArgs...); err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    // Fetch page
    listQuery, listArgs := buildListQuery(opts)
    var rows []userRow
    if err := r.db.SelectContext(ctx, &rows, listQuery, listArgs...); err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    users := make([]domain.User, len(rows))
    for i, row := range rows {
        users[i] = *row.toDomain()
    }
    return users, total, nil
}

func (r *sqlxUserRepository) Update(ctx context.Context, user *domain.User) error {
    result, err := r.db.NamedExecContext(ctx, updateUserSQL, map[string]any{
        "id":   user.ID,
        "name": user.Name,
        "role": user.Role,
    })
    if err != nil {
        return mapSqlxError(err)
    }
    rows, err := result.RowsAffected()
    if err != nil {
        return domain.ErrInternal.New(err)
    }
    if rows == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found", user.ID))
    }
    return nil
}

func (r *sqlxUserRepository) Delete(ctx context.Context, id string) error {
    result, err := r.db.ExecContext(ctx, softDeleteUserSQL, id)
    if err != nil {
        return domain.ErrInternal.New(err)
    }
    rows, err := result.RowsAffected()
    if err != nil {
        return domain.ErrInternal.New(err)
    }
    if rows == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found", id))
    }
    return nil
}

// mapSqlxError converts database/sql errors to domain errors.
func mapSqlxError(err error) error {
    if errors.Is(err, sql.ErrNoRows) {
        return domain.ErrNotFound.New(err)
    }
    // PostgreSQL unique violation (SQLSTATE 23505)
    if strings.Contains(err.Error(), "23505") {
        return domain.ErrConflict.New(err)
    }
    return domain.ErrInternal.New(err)
}

// newUUID generates a new UUID string.
// Use github.com/google/uuid in production.
func newUUID() string {
    return uuid.Must(uuid.NewV7()).String() // UUIDv7: time-sortable, better B-tree index performance
}

// buildCountQuery builds a parameterized COUNT query for the given options.
func buildCountQuery(opts domain.ListOptions) (string, []any) {
    conds := []string{"deleted_at IS NULL"}
    args := []any{}
    idx := 1
    if opts.Role != "" {
        conds = append(conds, fmt.Sprintf("role = $%d", idx))
        args = append(args, opts.Role)
    }
    return fmt.Sprintf("SELECT COUNT(*) FROM users WHERE %s", strings.Join(conds, " AND ")), args
}

// buildListQuery builds a parameterized SELECT with pagination.
func buildListQuery(opts domain.ListOptions) (string, []any) {
    conds := []string{"deleted_at IS NULL"}
    args := []any{}
    idx := 1
    if opts.Role != "" {
        conds = append(conds, fmt.Sprintf("role = $%d", idx))
        args = append(args, opts.Role)
        idx++
    }
    limit := opts.Limit
    if limit <= 0 || limit > 100 {
        limit = 20
    }
    offset := (opts.Page - 1) * limit
    args = append(args, limit, offset)
    query := fmt.Sprintf(`
        SELECT id, name, email, role, created_at, updated_at
        FROM users
        WHERE %s
        ORDER BY created_at DESC
        LIMIT $%d OFFSET $%d
    `, strings.Join(conds, " AND "), idx, idx+1)
    return query, args
}
```
