# GORM Patterns Reference

This file covers GORM-specific patterns for PostgreSQL integration in Gin APIs: model definition, CRUD operations, soft deletes, scopes, preloading, raw SQL, batch operations, hooks, connection pooling, error handling, and a complete repository implementation. Use when building the repository layer with GORM.

> **Architectural recommendation:** These are mainstream Go/GORM community patterns, not part of the Gin framework API.

## Table of Contents

1. [Model Definition](#model-definition)
2. [Connection Pooling](#connection-pooling)
3. [CRUD Operations](#crud-operations)
4. [Soft Deletes](#soft-deletes)
5. [Scopes](#scopes)
6. [Preloading Associations](#preloading-associations)
7. [Raw SQL](#raw-sql)
8. [Batch Operations](#batch-operations)
9. [Hooks — Trade-offs](#hooks--trade-offs)
10. [Error Handling](#error-handling)
11. [PostgreSQL-Specific Features](#postgresql-specific-features)
12. [Complete Repository Implementation](#complete-repository-implementation)

---

## Model Definition

GORM maps struct fields to columns via tags. Keep models in `internal/repository` (not `internal/domain`) to avoid leaking GORM into the domain layer.

```go
// internal/repository/models.go
package repository

import (
    "time"

    "github.com/google/uuid"
    "gorm.io/gorm"
    "myapp/internal/domain"
)

// UserModel is the GORM representation of the users table.
// It stays in the repository layer — convert to/from domain.User via methods.
type UserModel struct {
    ID           string         `gorm:"type:uuid;primaryKey;default:gen_random_uuid()"`
    Name         string         `gorm:"type:varchar(100);not null"`
    Email        string         `gorm:"type:varchar(255);uniqueIndex;not null"`
    PasswordHash string         `gorm:"type:varchar(255);not null"`
    Role         string         `gorm:"type:varchar(50);not null;default:'user'"`
    CreatedAt    time.Time      `gorm:"autoCreateTime"`
    UpdatedAt    time.Time      `gorm:"autoUpdateTime"`
    DeletedAt    gorm.DeletedAt `gorm:"index"` // enables soft delete
}

// TableName overrides the default table name.
func (UserModel) TableName() string { return "users" }

// toDomain converts a GORM model to the domain entity.
func (m *UserModel) ToDomain() *domain.User {
    return &domain.User{
        ID:        m.ID,
        Name:      m.Name,
        Email:     m.Email,
        Role:      m.Role,
        CreatedAt: m.CreatedAt,
        UpdatedAt: m.UpdatedAt,
    }
}

// fromDomain converts a domain entity to a GORM model.
func fromDomain(u *domain.User) *UserModel {
    return &UserModel{
        ID:    u.ID,
        Name:  u.Name,
        Email: u.Email,
        Role:  u.Role,
    }
}
```

**Why separate model and domain entity?** The domain layer must not import `gorm.io/gorm`. Separation lets you change GORM tags, add database-only fields (like `PasswordHash`), or swap ORMs without touching domain logic.

---

## Connection Pooling

```go
// internal/repository/db.go
package repository

import (
    "fmt"
    "log/slog"
    "strings"
    "time"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

type Config struct {
    DSN             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
    LogLevel        string // "silent" | "error" | "warn" | "info"
}

func NewGORMDB(cfg Config, appLogger *slog.Logger) (*gorm.DB, error) {
    gormCfg := &gorm.Config{
        Logger: logger.Default.LogMode(gormLogLevel(cfg.LogLevel)),
    }

    db, err := gorm.Open(postgres.Open(cfg.DSN), gormCfg)
    if err != nil {
        return nil, fmt.Errorf("gorm.Open: %w", err)
    }

    sqlDB, err := db.DB()
    if err != nil {
        return nil, fmt.Errorf("db.DB: %w", err)
    }

    sqlDB.SetMaxOpenConns(cfg.MaxOpenConns)       // concurrent DB connections
    sqlDB.SetMaxIdleConns(cfg.MaxIdleConns)       // idle connections kept alive
    sqlDB.SetConnMaxLifetime(cfg.ConnMaxLifetime)  // recycle connections

    if err := sqlDB.Ping(); err != nil {
        return nil, fmt.Errorf("db.Ping: %w", err)
    }

    appLogger.Info("database connected")
    return db, nil
}

func gormLogLevel(level string) logger.LogLevel {
    switch strings.ToLower(level) {
    case "info":
        return logger.Info
    case "warn":
        return logger.Warn
    case "error":
        return logger.Error
    default:
        return logger.Silent
    }
}
```

Recommended pool settings for most APIs:

| Setting           | Value          | Why                                    |
|-------------------|----------------|----------------------------------------|
| MaxOpenConns      | 25             | Limits PostgreSQL connections          |
| MaxIdleConns      | 5              | Keeps a warm pool without waste        |
| ConnMaxLifetime   | 5 minutes      | Prevents stale connections             |

---

## CRUD Operations

```go
// Create — sets CreatedAt/UpdatedAt automatically
func (r *gormUserRepository) Create(ctx context.Context, user *domain.User) error {
    m := fromDomain(user)
    if err := r.db.WithContext(ctx).Create(m).Error; err != nil {
        return domain.ErrInternal.New(err)
    }
    user.ID = m.ID // GORM sets the generated UUID back
    user.CreatedAt = m.CreatedAt
    user.UpdatedAt = m.UpdatedAt
    return nil
}

// GetByID — returns domain.ErrNotFound when row is missing
func (r *gormUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    var m UserModel
    if err := r.db.WithContext(ctx).First(&m, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return m.ToDomain(), nil
}

// Update — only updates non-zero fields with Save; use Updates for partial
func (r *gormUserRepository) Update(ctx context.Context, user *domain.User) error {
    result := r.db.WithContext(ctx).
        Model(&UserModel{}).
        Where("id = ?", user.ID).
        Updates(map[string]any{
            "name":       user.Name,
            "role":       user.Role,
            "updated_at": time.Now(),
        })
    if result.Error != nil {
        return domain.ErrInternal.New(result.Error)
    }
    if result.RowsAffected == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found", user.ID))
    }
    return nil
}

// Delete — soft delete if DeletedAt is in the model, hard delete otherwise
func (r *gormUserRepository) Delete(ctx context.Context, id string) error {
    result := r.db.WithContext(ctx).Delete(&UserModel{}, "id = ?", id)
    if result.Error != nil {
        return domain.ErrInternal.New(result.Error)
    }
    if result.RowsAffected == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found", id))
    }
    return nil
}
```

---

## Soft Deletes

Adding `gorm.DeletedAt` to the model activates soft deletes automatically. GORM adds `WHERE deleted_at IS NULL` to all queries.

```go
type UserModel struct {
    // ...
    DeletedAt gorm.DeletedAt `gorm:"index"` // soft delete column
}

// Delete sets deleted_at = NOW() instead of removing the row
r.db.Delete(&UserModel{}, "id = ?", id)

// Query ignores soft-deleted rows by default
r.db.Find(&users) // WHERE deleted_at IS NULL

// Include soft-deleted rows
r.db.Unscoped().Find(&users)

// Hard delete (permanently remove)
r.db.Unscoped().Delete(&UserModel{}, "id = ?", id)
```

**Trade-off:** Soft deletes keep audit history but require `Unscoped()` everywhere you intentionally query deleted records. Consider a dedicated `archived_users` table for large datasets to avoid index bloat.

---

## Scopes

Scopes encapsulate reusable query fragments.

```go
// internal/repository/scopes.go
package repository

import "gorm.io/gorm"

// ByRole filters users by role.
func ByRole(role string) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if role == "" {
            return db
        }
        return db.Where("role = ?", role)
    }
}

// Paginate applies offset/limit pagination.
func Paginate(page, limit int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if page < 1 {
            page = 1
        }
        if limit < 1 || limit > 100 {
            limit = 20
        }
        return db.Offset((page - 1) * limit).Limit(limit)
    }
}

// Usage in List:
func (r *gormUserRepository) List(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error) {
    var models []UserModel
    var total int64

    query := r.db.WithContext(ctx).Model(&UserModel{}).Scopes(ByRole(opts.Role))

    if err := query.Count(&total).Error; err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    if err := query.Scopes(Paginate(opts.Page, opts.Limit)).Find(&models).Error; err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    users := make([]domain.User, len(models))
    for i, m := range models {
        users[i] = *m.ToDomain()
    }
    return users, total, nil
}
```

---

## Preloading Associations

```go
// ProfileModel demonstrates a 1:1 association
type ProfileModel struct {
    ID     string `gorm:"type:uuid;primaryKey;default:gen_random_uuid()"`
    UserID string `gorm:"type:uuid;not null;uniqueIndex"`
    Bio    string `gorm:"type:text"`
}

type UserWithProfile struct {
    UserModel
    Profile ProfileModel `gorm:"foreignKey:UserID"`
}

// Preload loads Profile in a single additional query (N+1 safe)
func (r *gormUserRepository) GetWithProfile(ctx context.Context, id string) (*UserWithProfile, error) {
    var u UserWithProfile
    if err := r.db.WithContext(ctx).
        Preload("Profile").
        First(&u.UserModel, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return &u, nil
}

// Preload with condition
r.db.Preload("Orders", "status = ?", "active").Find(&users)
```

**Warning:** `Preload` issues one extra query per association, not one per row (GORM batches it). Still, avoid preloading large datasets — use `Joins` for filtering.

---

## Raw SQL

Use raw SQL for complex queries that GORM can't express cleanly.

```go
// Scan into a custom struct — not a GORM model
type UserStats struct {
    Role  string `gorm:"column:role"`
    Count int64  `gorm:"column:count"`
}

func (r *gormUserRepository) StatsByRole(ctx context.Context) ([]UserStats, error) {
    var stats []UserStats
    if err := r.db.WithContext(ctx).Raw(`
        SELECT role, COUNT(*) AS count
        FROM users
        WHERE deleted_at IS NULL
        GROUP BY role
        ORDER BY count DESC
    `).Scan(&stats).Error; err != nil {
        return nil, domain.ErrInternal.New(err)
    }
    return stats, nil
}

// Exec for mutations
func (r *gormUserRepository) BanUser(ctx context.Context, id string) error {
    result := r.db.WithContext(ctx).Exec(
        `UPDATE users SET role = 'banned', updated_at = NOW() WHERE id = ?`, id,
    )
    if result.Error != nil {
        return domain.ErrInternal.New(result.Error)
    }
    if result.RowsAffected == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found", id))
    }
    return nil
}
```

---

## Batch Operations

```go
// CreateInBatches — inserts in chunks to avoid parameter limit
func (r *gormUserRepository) BulkCreate(ctx context.Context, users []domain.User) error {
    models := make([]UserModel, len(users))
    for i, u := range users {
        models[i] = *fromDomain(&u)
    }
    // batchSize=100 — tune based on row size and DB limits
    if err := r.db.WithContext(ctx).CreateInBatches(models, 100).Error; err != nil {
        return domain.ErrInternal.New(err)
    }
    return nil
}

// FindInBatches — process large result sets without loading all rows into memory
func (r *gormUserRepository) ExportAll(ctx context.Context, process func([]domain.User) error) error {
    return r.db.WithContext(ctx).Model(&UserModel{}).
        FindInBatches(&[]UserModel{}, 200, func(tx *gorm.DB, batch int) error {
            var models []UserModel
            tx.Statement.Dest = &models
            users := make([]domain.User, len(models))
            for i, m := range models {
                users[i] = *m.ToDomain()
            }
            return process(users)
        }).Error
}
```

---

## Hooks — Trade-offs

GORM hooks (`BeforeCreate`, `AfterCreate`, etc.) run automatically but introduce hidden side effects.

```go
// Use hooks for truly cross-cutting concerns (e.g., UUID generation)
func (m *UserModel) BeforeCreate(tx *gorm.DB) error {
    if m.ID == "" {
        m.ID = uuid.Must(uuid.NewV7()).String() // UUIDv7: time-sortable, better B-tree index performance
    }
    return nil
}
```

**Trade-offs:**

| Pros | Cons |
|------|------|
| Zero boilerplate for UUID gen | Hidden logic — hard to trace |
| Consistent across all create paths | Can't be disabled per call |
| Works with transactions | Tested separately from service logic |

**Recommendation:** Use hooks for `ID` generation and `UpdatedAt` only. Move business logic (hashing passwords, sending emails) to the service layer where it's explicit and testable.

---

## Error Handling

Map GORM errors to domain errors. Never leak raw GORM errors to the handler layer.

```go
// internal/repository/errors.go
package repository

import (
    "errors"
    "strings"

    "gorm.io/gorm"
    "myapp/internal/domain"
)

// mapGORMError converts a GORM error to a domain error.
func mapGORMError(err error) error {
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return domain.ErrNotFound.New(err)
    }
    // PostgreSQL unique violation (error code 23505)
    if isUniqueViolation(err) {
        return domain.ErrConflict.New(err)
    }
    return domain.ErrInternal.New(err)
}

func isUniqueViolation(err error) bool {
    return err != nil && strings.Contains(err.Error(), "23505")
}

// Usage — consistent error mapping in every repository method:
func (r *gormUserRepository) Create(ctx context.Context, user *domain.User) error {
    m := fromDomain(user)
    if err := r.db.WithContext(ctx).Create(m).Error; err != nil {
        return mapGORMError(err)
    }
    user.ID = m.ID
    return nil
}
```

**Critical:** Log the raw GORM error at the service layer (it includes the SQL state). Return only the domain error to the handler.

---

## PostgreSQL-Specific Features

```go
// ON CONFLICT DO NOTHING — idempotent insert
r.db.WithContext(ctx).
    Clauses(clause.OnConflict{DoNothing: true}).
    Create(&m)

// ON CONFLICT DO UPDATE (upsert)
r.db.WithContext(ctx).
    Clauses(clause.OnConflict{
        Columns:   []clause.Column{{Name: "email"}},
        DoUpdates: clause.AssignmentColumns([]string{"name", "updated_at"}),
    }).
    Create(&m)

// RETURNING — get generated values after insert
r.db.WithContext(ctx).
    Clauses(clause.Returning{Columns: []clause.Column{{Name: "id"}, {Name: "created_at"}}}).
    Create(&m)

// Array type (requires github.com/lib/pq)
type UserModel struct {
    Tags pq.StringArray `gorm:"type:text[]"`
}

// JSONB column
type UserModel struct {
    Metadata datatypes.JSON `gorm:"type:jsonb"`
}
```

---

## Complete Repository Implementation

Full `UserRepository` satisfying the `domain.UserRepository` interface.

```go
// internal/repository/user_repository_gorm.go
package repository

import (
    "context"
    "errors"
    "fmt"
    "time"

    "gorm.io/gorm"
    "myapp/internal/domain"
)

type gormUserRepository struct {
    db *gorm.DB
}

// NewUserRepository returns a domain.UserRepository backed by GORM.
func NewUserRepository(db *gorm.DB) domain.UserRepository {
    return &gormUserRepository{db: db}
}

func (r *gormUserRepository) db(ctx context.Context) *gorm.DB {
    return r.db.WithContext(ctx)
}

func (r *gormUserRepository) Create(ctx context.Context, user *domain.User) error {
    m := fromDomain(user)
    if err := r.db.WithContext(ctx).Create(m).Error; err != nil {
        return mapGORMError(err)
    }
    user.ID = m.ID
    user.CreatedAt = m.CreatedAt
    user.UpdatedAt = m.UpdatedAt
    return nil
}

func (r *gormUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    var m UserModel
    if err := r.db.WithContext(ctx).First(&m, "id = ?", id).Error; err != nil {
        return nil, mapGORMError(err)
    }
    return m.ToDomain(), nil
}

func (r *gormUserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    var m UserModel
    if err := r.db.WithContext(ctx).First(&m, "email = ?", email).Error; err != nil {
        return nil, mapGORMError(err)
    }
    return m.ToDomain(), nil
}

func (r *gormUserRepository) List(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error) {
    var models []UserModel
    var total int64

    q := r.db.WithContext(ctx).Model(&UserModel{}).Scopes(ByRole(opts.Role))

    if err := q.Count(&total).Error; err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }
    if err := q.Scopes(Paginate(opts.Page, opts.Limit)).Find(&models).Error; err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    users := make([]domain.User, len(models))
    for i, m := range models {
        users[i] = *m.ToDomain()
    }
    return users, total, nil
}

func (r *gormUserRepository) Update(ctx context.Context, user *domain.User) error {
    result := r.db.WithContext(ctx).
        Model(&UserModel{}).
        Where("id = ?", user.ID).
        Updates(map[string]any{
            "name":       user.Name,
            "role":       user.Role,
            "updated_at": time.Now(),
        })
    if result.Error != nil {
        return mapGORMError(result.Error)
    }
    if result.RowsAffected == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found", user.ID))
    }
    return nil
}

func (r *gormUserRepository) Delete(ctx context.Context, id string) error {
    result := r.db.WithContext(ctx).Delete(&UserModel{}, "id = ?", id)
    if result.Error != nil {
        return domain.ErrInternal.New(result.Error)
    }
    if result.RowsAffected == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found", id))
    }
    return nil
}
```
