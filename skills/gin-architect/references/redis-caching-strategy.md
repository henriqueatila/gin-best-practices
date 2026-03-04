# Redis Caching Strategy Reference

Smart caching patterns for Go Gin APIs. Goes beyond basic get/set — covers what to cache, when to invalidate, how to prevent stampedes, and Redis beyond caching (sessions, distributed locks, pub/sub).

**Practical gate:** Use this file when `cross-cutting-concerns.md` basic patterns are not enough — multi-instance deployments, cache stampedes under load, or needing sessions/distributed locks.

---

## Table of Contents

1. [What to Cache (Decision Matrix)](#1-what-to-cache-decision-matrix)
2. [Cache-Aside vs Write-Through vs Write-Behind](#2-cache-aside-vs-write-through-vs-write-behind)
3. [Cache Key Design](#3-cache-key-design)
4. [Cache Stampede Prevention (singleflight)](#4-cache-stampede-prevention-singleflight)
5. [Cache Warming](#5-cache-warming)
6. [Pub/Sub Invalidation (Multi-Instance)](#6-pubsub-invalidation-multi-instance)
7. [Session Storage in Redis](#7-session-storage-in-redis)
8. [Distributed Locking](#8-distributed-locking)
9. [Redis Connection Setup](#9-redis-connection-setup)
10. [Cross-Skill References](#10-cross-skill-references)

---

## 1. What to Cache (Decision Matrix)

**Rule of thumb:** Cache data that is read-heavy, expensive to compute, and tolerable to be slightly stale.

| Data Type | Cache? | Strategy | TTL |
|---|---|---|---|
| User profile | Yes (read-heavy) | Cache-aside, invalidate on update | 10 min |
| Product catalog | Yes (read-heavy, shared) | Cache-aside + warming | 1 h |
| Session / auth tokens | Yes (every request) | Write-through to Redis | Token lifetime |
| Search results | Maybe (if query > 100ms) | Cache-aside | 5 min |
| Real-time inventory | No (stale = oversell) | Always query DB | — |
| Configuration / feature flags | Yes | Cache-aside + pub/sub invalidation | 5 min |
| Aggregated analytics | Yes (expensive) | Cache-aside | 15 min |
| One-time codes (OTP, CSRF) | Yes | Write-through, single-use delete | 5–15 min |

**Do NOT cache:**
- Data that must be consistent (payments, inventory counts)
- User-specific write-heavy data (cart in active checkout)
- Tiny datasets (<10 rows) — just query the DB

---

## 2. Cache-Aside vs Write-Through vs Write-Behind

### Cache-Aside (recommended default)

Read path: check cache → miss → fetch DB → store in cache → return.
Write path: update DB → invalidate (or update) cache.

```go
// 90% of use cases. Simple to reason about.
func (s *ProductService) GetByID(ctx context.Context, id string) (*Product, error) {
    key := fmt.Sprintf("myapp:product:%s", id)

    // 1. Try cache
    data, err := s.redis.Get(ctx, key).Bytes()
    if err == nil {
        var p Product
        if err := json.Unmarshal(data, &p); err == nil {
            return &p, nil
        }
    }
    if err != nil && err != redis.Nil {
        // Cache unavailable — degrade gracefully, go to DB
        s.logger.Warn("redis get failed", "key", key, "err", err)
    }

    // 2. Cache miss — fetch from DB
    p, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get product: %w", err)
    }

    // 3. Store in cache (fire-and-forget, don't fail request on cache error)
    if b, err := json.Marshal(p); err == nil {
        _ = s.redis.Set(ctx, key, b, time.Hour).Err()
    }

    return p, nil
}

func (s *ProductService) Update(ctx context.Context, p *Product) error {
    if err := s.repo.Update(ctx, p); err != nil {
        return fmt.Errorf("update product: %w", err)
    }
    // Invalidate — simpler and safer than update
    key := fmt.Sprintf("myapp:product:%s", p.ID)
    _ = s.redis.Del(ctx, key).Err()
    return nil
}
```

### Write-Through

Write to cache and DB atomically on every write. Cache always warm. Adds write latency.

```go
func (s *SessionService) Save(ctx context.Context, sess *Session) error {
    // Write to DB first
    if err := s.repo.Save(ctx, sess); err != nil {
        return fmt.Errorf("save session: %w", err)
    }

    // Write to Redis synchronously
    b, err := json.Marshal(sess)
    if err != nil {
        return fmt.Errorf("marshal session: %w", err)
    }
    key := fmt.Sprintf("myapp:session:%s", sess.ID)
    if err := s.redis.Set(ctx, key, b, sess.TTL()).Err(); err != nil {
        // Log but don't fail — DB is source of truth
        s.logger.Error("redis write-through failed", "key", key, "err", err)
    }
    return nil
}
```

### Write-Behind (async)

Write to cache immediately, flush to DB asynchronously. High performance, risk of data loss on crash. Use only for non-critical counters (view counts, likes).

```go
// Not recommended for critical data. Sketch only.
func (s *StatsService) IncrView(ctx context.Context, postID string) {
    key := fmt.Sprintf("myapp:views:%s", postID)
    s.redis.Incr(ctx, key) // instant
    // Background worker flushes to DB every 30s
}
```

---

## 3. Cache Key Design

**Pattern:** `{app}:{version}:{entity}:{id}` or `{app}:{entity}:{qualifier}`

```go
// Good key patterns
const (
    KeyUser       = "myapp:v1:user:%s"           // myapp:v1:user:123
    KeyUserList   = "myapp:v1:user:list:%d:%d"   // page:size
    KeyProduct    = "myapp:v1:product:%s"
    KeySession    = "myapp:session:%s"            // no version — raw token
    KeyFeatureFlag = "myapp:config:flags"         // single hash
)

func userKey(id string) string {
    return fmt.Sprintf(KeyUser, id)
}

func userListKey(page, size int) string {
    return fmt.Sprintf(KeyUserList, page, size)
}
```

**Rules:**
- Namespace prevents key collision between services sharing one Redis
- Version (`v1`, `v2`) allows breaking schema changes without cache flush
- Never interpolate raw user input — normalize and validate first
- Keep keys short but readable; Redis memory is per-key metadata too
- Use `SCAN` + namespace prefix to bulk-delete (e.g., invalidate all user list pages)

```go
// Bulk invalidate list cache for a user namespace
func (s *UserService) InvalidateListCache(ctx context.Context) error {
    var cursor uint64
    for {
        keys, next, err := s.redis.Scan(ctx, cursor, "myapp:v1:user:list:*", 100).Result()
        if err != nil {
            return fmt.Errorf("scan keys: %w", err)
        }
        if len(keys) > 0 {
            _ = s.redis.Del(ctx, keys...).Err()
        }
        cursor = next
        if cursor == 0 {
            break
        }
    }
    return nil
}
```

---

## 4. Cache Stampede Prevention (singleflight)

**Problem:** Cache TTL expires. 1000 concurrent requests all get cache miss → all hit DB → DB overloads.

**Solution:** `singleflight` ensures only ONE goroutine fetches from DB; others wait and share the result.

```go
import (
    "golang.org/x/sync/singleflight"
)

type ProductService struct {
    redis  *redis.Client
    repo   ProductRepository
    sf     singleflight.Group
    logger *slog.Logger
}

func (s *ProductService) GetByID(ctx context.Context, id string) (*Product, error) {
    key := fmt.Sprintf("myapp:v1:product:%s", id)

    // 1. Try cache
    data, err := s.redis.Get(ctx, key).Bytes()
    if err == nil {
        var p Product
        if jsonErr := json.Unmarshal(data, &p); jsonErr == nil {
            return &p, nil
        }
    }

    // 2. Deduplicate concurrent fetches for same key
    v, err, _ := s.sf.Do(key, func() (any, error) {
        // Only one goroutine reaches here per key
        p, err := s.repo.GetByID(ctx, id)
        if err != nil {
            return nil, fmt.Errorf("fetch product %s: %w", id, err)
        }

        b, _ := json.Marshal(p)
        ttl := time.Hour + time.Duration(rand.Intn(300))*time.Second // jitter
        _ = s.redis.Set(ctx, key, b, ttl).Err()

        return p, nil
    })
    if err != nil {
        return nil, err
    }

    return v.(*Product), nil
}
```

**TTL jitter:** Add random seconds to TTL so cached items expire at different times — prevents synchronized stampedes on bulk-loaded data.

```go
func withJitter(base time.Duration, maxJitterSec int) time.Duration {
    return base + time.Duration(rand.Intn(maxJitterSec))*time.Second
}
// Usage: ttl := withJitter(time.Hour, 300) // 60–65 min
```

---

## 5. Cache Warming

**Problem:** Cold start (deploy, flush) causes stampede before cache is populated.

**Pattern A — Startup warming** for critical, bounded datasets:

```go
func (s *ProductService) WarmCache(ctx context.Context) error {
    s.logger.Info("warming product cache")
    products, err := s.repo.ListAll(ctx) // Only feasible for small catalogs
    if err != nil {
        return fmt.Errorf("warm cache: %w", err)
    }

    pipe := s.redis.Pipeline()
    for _, p := range products {
        b, _ := json.Marshal(p)
        key := fmt.Sprintf("myapp:v1:product:%s", p.ID)
        pipe.Set(ctx, key, b, withJitter(time.Hour, 300))
    }
    _, err = pipe.Exec(ctx)
    return err
}

// Call in main.go after Redis connection established
func main() {
    // ... setup ...
    if err := productService.WarmCache(context.Background()); err != nil {
        logger.Warn("cache warm failed, degrading gracefully", "err", err)
    }
    // Start server
}
```

**Pattern B — Background refresh** for large datasets (refresh before expiry):

```go
// Cache item stores both data and refresh-at timestamp
type CachedItem[T any] struct {
    Data      T         `json:"data"`
    ExpiresAt time.Time `json:"expires_at"`
    RefreshAt time.Time `json:"refresh_at"` // 80% of TTL
}

func (s *ProductService) GetWithEarlyRefresh(ctx context.Context, id string) (*Product, error) {
    key := fmt.Sprintf("myapp:v1:product:%s", id)

    data, err := s.redis.Get(ctx, key).Bytes()
    if err == nil {
        var item CachedItem[Product]
        if json.Unmarshal(data, &item) == nil {
            if time.Now().After(item.RefreshAt) {
                // Trigger async refresh — serve stale, update in background
                go s.refreshAsync(context.Background(), id, key)
            }
            return &item.Data, nil
        }
    }

    return s.fetchAndCache(ctx, id, key)
}

func (s *ProductService) refreshAsync(ctx context.Context, id, key string) {
    _, _, _ = s.sf.Do("refresh:"+key, func() (any, error) {
        return s.fetchAndCache(ctx, id, key)
    })
}
```

---

## 6. Pub/Sub Invalidation (Multi-Instance)

**When to use:** Multiple app instances each with local in-memory cache (e.g., `sync.Map`, `ristretto`). When instance A updates data, it must tell other instances to invalidate their local cache.

```go
const invalidationChannel = "myapp:cache:invalidations"

type InvalidationMessage struct {
    Entity string `json:"entity"` // "product", "user"
    ID     string `json:"id"`
}

// Publisher — called on write
func (s *ProductService) publishInvalidation(ctx context.Context, id string) {
    msg := InvalidationMessage{Entity: "product", ID: id}
    b, _ := json.Marshal(msg)
    if err := s.redis.Publish(ctx, invalidationChannel, b).Err(); err != nil {
        s.logger.Warn("publish invalidation failed", "err", err)
    }
}

// Subscriber — started at app boot, runs in goroutine
func (s *ProductService) SubscribeInvalidations(ctx context.Context) {
    sub := s.redis.Subscribe(ctx, invalidationChannel)
    defer sub.Close()

    ch := sub.Channel()
    for {
        select {
        case <-ctx.Done():
            return
        case msg, ok := <-ch:
            if !ok {
                return
            }
            var inv InvalidationMessage
            if err := json.Unmarshal([]byte(msg.Payload), &inv); err != nil {
                continue
            }
            if inv.Entity == "product" {
                s.localCache.Delete(inv.ID) // evict from in-process cache
            }
        }
    }
}

// Wire in main.go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    go productService.SubscribeInvalidations(ctx)
    // ... start server
}
```

---

## 7. Session Storage in Redis

**When to use:** Server-side sessions (JWT revocation, large session payloads, multi-device logout).

```go
type SessionStore struct {
    redis  *redis.Client
    logger *slog.Logger
}

type UserSession struct {
    UserID    string    `json:"user_id"`
    Email     string    `json:"email"`
    Roles     []string  `json:"roles"`
    CreatedAt time.Time `json:"created_at"`
}

func sessionKey(sessionID string) string {
    return fmt.Sprintf("myapp:session:%s", sessionID)
}

func (s *SessionStore) Save(ctx context.Context, sessionID string, sess *UserSession, ttl time.Duration) error {
    b, err := json.Marshal(sess)
    if err != nil {
        return fmt.Errorf("marshal session: %w", err)
    }
    if err := s.redis.Set(ctx, sessionKey(sessionID), b, ttl).Err(); err != nil {
        return fmt.Errorf("save session: %w", err)
    }
    return nil
}

func (s *SessionStore) Get(ctx context.Context, sessionID string) (*UserSession, error) {
    data, err := s.redis.Get(ctx, sessionKey(sessionID)).Bytes()
    if err == redis.Nil {
        return nil, nil // session not found / expired
    }
    if err != nil {
        return nil, fmt.Errorf("get session: %w", err)
    }
    var sess UserSession
    if err := json.Unmarshal(data, &sess); err != nil {
        return nil, fmt.Errorf("unmarshal session: %w", err)
    }
    return &sess, nil
}

func (s *SessionStore) Delete(ctx context.Context, sessionID string) error {
    return s.redis.Del(ctx, sessionKey(sessionID)).Err()
}

// Gin middleware using session store
func SessionMiddleware(store *SessionStore) gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "missing token"})
            return
        }

        sess, err := store.Get(c.Request.Context(), token)
        if err != nil || sess == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid session"})
            return
        }

        c.Set("userID", sess.UserID)
        c.Set("roles", sess.Roles)
        c.Next()
    }
}
```

---

## 8. Distributed Locking

**When to use:** Prevent duplicate processing in multi-instance deployments (duplicate payment, double-send email, concurrent inventory update).

```go
type DistributedLock struct {
    redis *redis.Client
}

// TryLock acquires lock. Returns (true, nil) if acquired. Returns (false, nil) if already locked.
func (l *DistributedLock) TryLock(ctx context.Context, resource string, ttl time.Duration) (acquired bool, unlock func(), err error) {
    lockKey := fmt.Sprintf("myapp:lock:%s", resource)
    lockVal := fmt.Sprintf("%d", time.Now().UnixNano()) // unique value per acquisition

    ok, err := l.redis.SetNX(ctx, lockKey, lockVal, ttl).Result()
    if err != nil {
        return false, nil, fmt.Errorf("acquire lock %s: %w", resource, err)
    }
    if !ok {
        return false, nil, nil // already locked
    }

    unlock = func() {
        // Only delete if we still own it (value check prevents deleting another holder's lock)
        script := redis.NewScript(`
            if redis.call("get", KEYS[1]) == ARGV[1] then
                return redis.call("del", KEYS[1])
            else
                return 0
            end
        `)
        _ = script.Run(ctx, l.redis, []string{lockKey}, lockVal).Err()
    }

    return true, unlock, nil
}

// Usage — prevent duplicate payment processing
func (s *PaymentService) ProcessPayment(ctx context.Context, orderID string) error {
    acquired, unlock, err := s.lock.TryLock(ctx, "payment:"+orderID, 30*time.Second)
    if err != nil {
        return fmt.Errorf("lock check: %w", err)
    }
    if !acquired {
        return fmt.Errorf("payment for order %s already in progress", orderID)
    }
    defer unlock()

    return s.doProcessPayment(ctx, orderID)
}
```

**Notes:**
- TTL must exceed max expected operation duration
- The Lua script ensures atomic check-and-delete (no race between check and del)
- For stronger guarantees (Redlock), use `github.com/go-redsync/redsync`

---

## 9. Redis Connection Setup

```go
import (
    "context"
    "fmt"
    "log/slog"
    "time"

    "github.com/redis/go-redis/v9"
)

func NewRedisClient(url string, logger *slog.Logger) (*redis.Client, error) {
    opts, err := redis.ParseURL(url) // e.g. redis://user:pass@localhost:6379/0
    if err != nil {
        return nil, fmt.Errorf("parse redis url: %w", err)
    }

    // Tune pool for API server load
    opts.PoolSize = 20              // max connections per instance
    opts.MinIdleConns = 5          // keep warm
    opts.ConnMaxIdleTime = 5 * time.Minute
    opts.DialTimeout = 3 * time.Second
    opts.ReadTimeout = 2 * time.Second
    opts.WriteTimeout = 2 * time.Second

    client := redis.NewClient(opts)

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := client.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("redis ping: %w", err)
    }

    logger.Info("redis connected", "addr", opts.Addr)
    return client, nil
}
```

**Graceful degradation pattern** — cache miss is not a failure:

```go
// Wrap Redis calls so cache errors don't fail requests
func safeGet(ctx context.Context, rdb *redis.Client, key string) ([]byte, bool) {
    data, err := rdb.Get(ctx, key).Bytes()
    if err == redis.Nil {
        return nil, false // clean miss
    }
    if err != nil {
        // Redis unavailable — treat as miss, log, continue
        slog.WarnContext(ctx, "redis get error", "key", key, "err", err)
        return nil, false
    }
    return data, true
}
```

**Required Go modules:**

```bash
go get github.com/redis/go-redis/v9
go get golang.org/x/sync
```

---

## 10. Cross-Skill References

| Topic | File |
|---|---|
| Basic Redis get/set/TTL patterns | `cross-cutting-concerns.md` → Redis Cache section |
| Circuit breaker around Redis calls | `resilience-patterns.md` → Circuit Breaker section |
| Health check endpoint for Redis | `cross-cutting-concerns.md` → Health Checks |
| Deployment: Redis config, sentinel, cluster | `gin-deploy` skill (separate) |
| Structured logging with slog | `cross-cutting-concerns.md` → Logging section |
