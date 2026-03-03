# Rate Limiting Reference

This file covers production rate limiting strategies for Gin APIs: in-memory token bucket (single-node), sliding window counter, Redis-backed distributed limiting (token bucket and sliding window), per-user/API-key quotas, tiered limits, standard response headers, and graceful Redis degradation. See `middleware.md` for a simpler in-memory recap.

## Table of Contents

1. [Algorithm Overview](#algorithm-overview)
2. [In-Memory Token Bucket](#in-memory-token-bucket)
3. [Sliding Window Counter](#sliding-window-counter)
4. [Redis Token Bucket (Lua)](#redis-token-bucket-lua)
5. [Redis Sliding Window](#redis-sliding-window)
6. [Per-User / API-Key Limiting](#per-user--api-key-limiting)
7. [Tiered Limits](#tiered-limits)
8. [Response Headers](#response-headers)
9. [Graceful Degradation](#graceful-degradation)

---

## Algorithm Overview

| Algorithm | Accuracy | Memory | Distributed | Best For |
|---|---|---|---|---|
| Fixed window | Low (boundary burst) | O(1) | Yes (INCR) | Simple global caps |
| Token bucket | High | O(clients) | Yes (Lua) | Smooth traffic shaping |
| Sliding window counter | Medium-high | O(1) | Yes (INCR×2) | Accurate without per-request storage |
| Sliding window log | Exact | O(requests) | Yes (ZADD) | High-value APIs, low traffic |

**Token bucket** (via `golang.org/x/time/rate`) is best for single-node deployments — it allows short bursts while enforcing a long-term rate. **Sliding window counter** blends two fixed-window buckets for near-exact counts without storing individual timestamps. **Redis Lua** scripts ensure atomic check-and-decrement across instances.

Use **in-memory** when: single binary, simplicity matters, losing counts on restart is acceptable.
Use **Redis** when: multiple instances, counts must survive restarts, per-user billing accuracy required.

---

## In-Memory Token Bucket

`golang.org/x/time/rate` wraps the token bucket algorithm. This is an improved version of the snippet in `middleware.md` — it accepts a configurable cleanup interval and a done channel for clean shutdown.

```go
// pkg/middleware/rate_limiter_memory.go
package middleware

import (
	"log/slog"
	"net/http"
	"sync"
	"time"

	"github.com/gin-gonic/gin"
	"golang.org/x/time/rate"
)

type clientLimiter struct {
	limiter  *rate.Limiter
	lastSeen time.Time
}

// MemoryRateLimiterConfig configures the in-memory rate limiter.
type MemoryRateLimiterConfig struct {
	// Rate is the number of requests allowed per second.
	Rate rate.Limit
	// Burst is the maximum number of requests allowed in a single burst.
	Burst int
	// CleanupInterval controls how often stale entries are removed.
	CleanupInterval time.Duration
	// TTL is how long a client entry lives without activity before removal.
	TTL time.Duration
}

// MemoryRateLimiter limits requests per client IP using a token bucket.
// It is NOT suitable for multi-instance deployments — use RedisRateLimiter instead.
func MemoryRateLimiter(cfg MemoryRateLimiterConfig, done <-chan struct{}) gin.HandlerFunc {
	if cfg.CleanupInterval == 0 {
		cfg.CleanupInterval = time.Minute
	}
	if cfg.TTL == 0 {
		cfg.TTL = 3 * time.Minute
	}

	var mu sync.Mutex
	limiters := make(map[string]*clientLimiter)

	// Background cleanup — exits when done is closed.
	go func() {
		ticker := time.NewTicker(cfg.CleanupInterval)
		defer ticker.Stop()
		for {
			select {
			case <-ticker.C:
				mu.Lock()
				for ip, cl := range limiters {
					if time.Since(cl.lastSeen) > cfg.TTL {
						delete(limiters, ip)
					}
				}
				mu.Unlock()
			case <-done:
				return
			}
		}
	}()

	getLimiter := func(ip string) *rate.Limiter {
		mu.Lock()
		defer mu.Unlock()
		cl, ok := limiters[ip]
		if !ok {
			cl = &clientLimiter{limiter: rate.NewLimiter(cfg.Rate, cfg.Burst)}
			limiters[ip] = cl
		}
		cl.lastSeen = time.Now()
		return cl.limiter
	}

	return func(c *gin.Context) {
		ip := c.ClientIP()
		lim := getLimiter(ip)
		if !lim.Allow() {
			slog.WarnContext(c.Request.Context(), "rate limit exceeded", "ip", ip)
			c.Header("Retry-After", "1")
			c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
				"error": "rate limit exceeded",
			})
			return
		}
		c.Next()
	}
}
```

**Usage:**

```go
done := make(chan struct{})
defer close(done)

r.Use(middleware.MemoryRateLimiter(middleware.MemoryRateLimiterConfig{
    Rate:  10,
    Burst: 20,
}, done))
```

**Why `done` channel:** The goroutine would leak if the server restarts or the router is rebuilt in tests. Closing `done` stops the ticker cleanly.

---

## Sliding Window Counter

Blends two consecutive fixed-window buckets weighted by elapsed time in the current window. More accurate than a plain fixed window (no boundary burst) without storing per-request timestamps.

```go
// pkg/middleware/rate_limiter_sliding.go
package middleware

import (
	"log/slog"
	"net/http"
	"sync"
	"time"

	"github.com/gin-gonic/gin"
)

type windowBucket struct {
	count    int
	windowAt time.Time // start of the current window
}

// SlidingWindowLimiter limits requests using a sliding window counter.
// windowSize: duration of one window (e.g. time.Minute).
// limit: max requests per window.
func SlidingWindowLimiter(windowSize time.Duration, limit int) gin.HandlerFunc {
	var mu sync.Mutex
	buckets := make(map[string][2]windowBucket) // [prev, curr]

	return func(c *gin.Context) {
		key := c.ClientIP()
		now := time.Now()
		windowStart := now.Truncate(windowSize)

		mu.Lock()
		pair := buckets[key]

		// Advance windows if necessary.
		if pair[1].windowAt != windowStart {
			if pair[1].windowAt.Add(windowSize).Equal(windowStart) {
				// Shift: current → previous.
				pair[0] = pair[1]
			} else {
				// Gap larger than one window — reset both.
				pair[0] = windowBucket{}
			}
			pair[1] = windowBucket{windowAt: windowStart}
		}

		// Weighted count: fraction of previous window that overlaps current.
		elapsed := now.Sub(windowStart)
		overlap := 1.0 - elapsed.Seconds()/windowSize.Seconds()
		count := int(float64(pair[0].count)*overlap) + pair[1].count

		if count >= limit {
			mu.Unlock()
			slog.WarnContext(c.Request.Context(), "sliding window limit exceeded", "ip", key)
			c.Header("Retry-After", windowSize.String())
			c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
				"error": "rate limit exceeded",
			})
			return
		}

		pair[1].count++
		buckets[key] = pair
		mu.Unlock()

		c.Next()
	}
}
```

**Why weighted overlap:** If a client sent 90 requests in the last window and the current window is 30% complete, the effective count is `90 * 0.7 + current`. This prevents the burst that occurs at fixed-window boundaries.

---

## Redis Token Bucket (Lua)

A Lua script executes atomically on the Redis instance — no race between read and write. Uses two keys per client: `tokens` (current count) and `ts` (last refill timestamp).

```go
// pkg/middleware/rate_limiter_redis.go
package middleware

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/redis/go-redis/v9"
)

// tokenBucketScript atomically checks and decrements a token bucket stored in Redis.
// KEYS[1]: token count key, KEYS[2]: last-refill timestamp key
// ARGV[1]: capacity, ARGV[2]: refill rate (tokens/sec), ARGV[3]: current unix timestamp (float)
// Returns: remaining tokens after the request (-1 means denied).
var tokenBucketScript = redis.NewScript(`
local tokens_key = KEYS[1]
local ts_key     = KEYS[2]
local capacity   = tonumber(ARGV[1])
local rate       = tonumber(ARGV[2])
local now        = tonumber(ARGV[3])
local ttl        = math.ceil(capacity / rate) + 1

local last_ts = tonumber(redis.call("GET", ts_key))
local tokens

if last_ts == nil then
    tokens = capacity
else
    local elapsed = now - last_ts
    tokens = math.min(capacity, tonumber(redis.call("GET", tokens_key) or 0) + elapsed * rate)
end

if tokens < 1 then
    return -1
end

tokens = tokens - 1
redis.call("SETEX", tokens_key, ttl, tokens)
redis.call("SETEX", ts_key, ttl, now)
return tokens
`)

// RedisTokenBucketConfig configures the Redis-backed token bucket limiter.
type RedisTokenBucketConfig struct {
	Client   *redis.Client
	Capacity int           // maximum tokens (burst size)
	Rate     float64       // tokens refilled per second
	KeyPrefix string       // e.g. "rl:api:"
}

// RedisTokenBucketLimiter limits requests using an atomic token bucket in Redis.
func RedisTokenBucketLimiter(cfg RedisTokenBucketConfig) gin.HandlerFunc {
	return func(c *gin.Context) {
		ctx := c.Request.Context()
		ip := c.ClientIP()
		base := cfg.KeyPrefix + ip
		tokensKey := base + ":tokens"
		tsKey := base + ":ts"
		now := float64(time.Now().UnixNano()) / 1e9

		remaining, err := tokenBucketScript.Run(ctx, cfg.Client,
			[]string{tokensKey, tsKey},
			cfg.Capacity, cfg.Rate, now,
		).Int()

		if err != nil && !errors.Is(err, redis.Nil) {
			// Redis error — log and allow (fail open). See Graceful Degradation section.
			slog.ErrorContext(ctx, "redis rate limiter error", "err", err)
			c.Next()
			return
		}

		c.Header("X-RateLimit-Limit", strconv.Itoa(cfg.Capacity))

		if remaining < 0 {
			c.Header("X-RateLimit-Remaining", "0")
			c.Header("Retry-After", strconv.Itoa(int(1.0/cfg.Rate)+1))
			c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
				"error": "rate limit exceeded",
			})
			return
		}

		c.Header("X-RateLimit-Remaining", strconv.Itoa(remaining))
		c.Next()
	}
}
```

**Why Lua:** `GET` + conditional `SET` in application code has a TOCTOU race under concurrent requests. A Lua script is executed atomically by the Redis server — no other command runs between the read and write.

---

## Redis Sliding Window

Uses a sorted set where each member is a unique request ID and the score is the Unix timestamp (nanoseconds). `ZREMRANGEBYSCORE` prunes old entries; `ZCARD` counts the current window.

```go
// pkg/middleware/rate_limiter_redis_sliding.go
package middleware

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/redis/go-redis/v9"
)

// redisSlidingScript atomically records a request and checks the window count.
// KEYS[1]: sorted set key
// ARGV[1]: window size in nanoseconds, ARGV[2]: current timestamp (ns), ARGV[3]: limit, ARGV[4]: unique request ID
// Returns: 0 = allowed, 1 = denied.
var redisSlidingScript = redis.NewScript(`
local key    = KEYS[1]
local window = tonumber(ARGV[1])
local now    = tonumber(ARGV[2])
local limit  = tonumber(ARGV[3])
local req_id = ARGV[4]
local cutoff = now - window

redis.call("ZREMRANGEBYSCORE", key, "-inf", cutoff)
local count = redis.call("ZCARD", key)

if count >= limit then
    return 1
end

redis.call("ZADD", key, now, req_id)
redis.call("PEXPIRE", key, math.ceil(window / 1e6))  -- ns → ms
return 0
`)

// RedisSlidingWindowConfig configures the Redis sliding window limiter.
type RedisSlidingWindowConfig struct {
	Client    *redis.Client
	Window    time.Duration
	Limit     int
	KeyPrefix string
}

// RedisSlidingWindowLimiter limits requests using a Redis sorted-set sliding window.
func RedisSlidingWindowLimiter(cfg RedisSlidingWindowConfig) gin.HandlerFunc {
	windowNs := cfg.Window.Nanoseconds()

	return func(c *gin.Context) {
		ctx := c.Request.Context()
		ip := c.ClientIP()
		key := cfg.KeyPrefix + ip
		now := time.Now().UnixNano()
		// Unique member per request to avoid ZADD deduplication.
		reqID := strconv.FormatInt(now, 36) + ip

		denied, err := redisSlidingScript.Run(ctx, cfg.Client,
			[]string{key},
			windowNs, now, cfg.Limit, reqID,
		).Int()

		if err != nil && !errors.Is(err, redis.Nil) {
			slog.ErrorContext(ctx, "redis sliding window error", "err", err)
			c.Next() // fail open
			return
		}

		c.Header("X-RateLimit-Limit", strconv.Itoa(cfg.Limit))

		if denied == 1 {
			c.Header("X-RateLimit-Remaining", "0")
			c.Header("Retry-After", strconv.FormatFloat(cfg.Window.Seconds(), 'f', 0, 64))
			c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
				"error": "rate limit exceeded",
			})
			return
		}

		c.Next()
	}
}
```

**Memory trade-off:** Each request is a sorted set entry. For 1000 req/min limit × 10 000 clients = ~10M entries. At ~100 bytes/entry this is ~1 GB. Use the sliding window **counter** approach (two INCR keys) if memory is a concern at scale.

---

## Per-User / API-Key Limiting

Replace the IP key with a user ID from JWT claims or an `X-API-Key` header. This survives IP changes (mobile clients) and is harder to spoof than IP.

```go
// pkg/middleware/rate_limiter_key_extractor.go
package middleware

import (
	"net/http"
	"strings"

	"github.com/gin-gonic/gin"
	"github.com/golang-jwt/jwt/v5"
)

// ClientKeyFunc extracts a rate-limit key from the request.
// Returning "" falls back to ClientIP.
type ClientKeyFunc func(c *gin.Context) string

// APIKeyExtractor reads the X-API-Key header.
func APIKeyExtractor(c *gin.Context) string {
	if key := c.GetHeader("X-API-Key"); key != "" {
		return "apikey:" + key
	}
	return ""
}

// JWTSubjectExtractor reads the "sub" claim from a Bearer token.
// It does NOT validate the token — pair with AuthRequired middleware first.
func JWTSubjectExtractor(c *gin.Context) string {
	header := c.GetHeader("Authorization")
	if !strings.HasPrefix(header, "Bearer ") {
		return ""
	}
	tokenStr := strings.TrimPrefix(header, "Bearer ")
	// Parse without validation — signature already verified by AuthRequired.
	claims := jwt.MapClaims{}
	parser := jwt.NewParser()
	token, _, err := parser.ParseUnverified(tokenStr, claims)
	if err != nil {
		return ""
	}
	_ = token // ParseUnverified skips validation — token.Valid is always false
	sub, err := claims.GetSubject()
	if err != nil || sub == "" {
		return ""
	}
	return "user:" + sub
}

// KeyedRateLimiter wraps any limiter middleware with a custom key extractor.
// keyFn derives the rate-limit key; if it returns "", c.ClientIP() is used.
// limiterFn must accept a key and return gin.HandlerFunc (partial application pattern).
func withKeyExtractor(keyFn ClientKeyFunc, next func(key string) gin.HandlerFunc) gin.HandlerFunc {
	return func(c *gin.Context) {
		key := keyFn(c)
		if key == "" {
			key = c.ClientIP()
		}
		next(key)(c)
	}
}
```

**Usage with Redis token bucket:**

```go
// main.go
limiter := func(key string) gin.HandlerFunc {
    return middleware.RedisTokenBucketLimiter(middleware.RedisTokenBucketConfig{
        Client:    rdb,
        Capacity:  100,
        Rate:      10,
        KeyPrefix: key + ":",
    })
}

r.Use(middleware.APIKeyExtractor) // sets key; shown simplified here
// In practice, compose withKeyExtractor + limiter at route group level.
```

---

## Tiered Limits

Different limits per user role or subscription plan. Load from config; avoid hardcoding.

```go
// pkg/middleware/rate_limiter_tiered.go
package middleware

import (
	"log/slog"
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/redis/go-redis/v9"
	"golang.org/x/time/rate"
)

// TierConfig holds rate limit parameters for one tier.
type TierConfig struct {
	Rate  rate.Limit // requests per second (in-memory) or tokens/sec (Redis)
	Burst int        // max burst
}

// DefaultTiers maps role names to limits. Override from env/config.
var DefaultTiers = map[string]TierConfig{
	"anonymous":     {Rate: 2, Burst: 5},
	"authenticated": {Rate: 20, Burst: 40},
	"premium":       {Rate: 100, Burst: 200},
}

// TieredRateLimiter applies per-role limits using the Redis token bucket.
// It reads "role" from the Gin context (set by AuthRequired middleware).
func TieredRateLimiter(rdb *redis.Client, tiers map[string]TierConfig, keyPrefix string) gin.HandlerFunc {
	fallback := tiers["anonymous"]

	return func(c *gin.Context) {
		ctx := c.Request.Context()

		role, _ := c.Get("role") // set by gin-auth AuthRequired
		roleStr, _ := role.(string)
		cfg, ok := tiers[roleStr]
		if !ok {
			cfg = fallback
		}

		sub, _ := c.Get("sub") // user ID from JWT
		subStr, _ := sub.(string)
		if subStr == "" {
			subStr = c.ClientIP()
		}
		base := keyPrefix + roleStr + ":" + subStr

		now := float64(time.Now().UnixNano()) / 1e9
		remaining, err := tokenBucketScript.Run(ctx, rdb,
			[]string{base + ":tokens", base + ":ts"},
			cfg.Burst, float64(cfg.Rate), now,
		).Int()

		if err != nil && err != redis.Nil {
			slog.ErrorContext(ctx, "tiered limiter redis error", "err", err, "role", roleStr)
			c.Next()
			return
		}

		c.Header("X-RateLimit-Limit", strconv.Itoa(cfg.Burst))

		if remaining < 0 {
			c.Header("X-RateLimit-Remaining", "0")
			retryAfter := int(1.0/float64(cfg.Rate)) + 1
			c.Header("Retry-After", strconv.Itoa(retryAfter))
			c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
				"error": "rate limit exceeded",
			})
			return
		}

		c.Header("X-RateLimit-Remaining", strconv.Itoa(remaining))
		c.Next()
	}
}
```

**Loading tiers from environment:**

```go
// config/rate_limits.go
package config

import (
	"os"
	"strconv"

	"golang.org/x/time/rate"
)

func LoadTiers() map[string]middleware.TierConfig {
	tiers := make(map[string]middleware.TierConfig)
	for _, role := range []string{"anonymous", "authenticated", "premium"} {
		rateKey := "RATE_" + strings.ToUpper(role) + "_RPS"
		burstKey := "RATE_" + strings.ToUpper(role) + "_BURST"
		r, err := strconv.ParseFloat(os.Getenv(rateKey), 64)
		if err != nil {
			r = float64(middleware.DefaultTiers[role].Rate)
		}
		b, err := strconv.Atoi(os.Getenv(burstKey))
		if err != nil {
			b = middleware.DefaultTiers[role].Burst
		}
		tiers[role] = middleware.TierConfig{Rate: rate.Limit(r), Burst: b}
	}
	return tiers
}
```

---

## Response Headers

Set these on every response — not only on 429 — so clients can self-throttle before hitting the limit.

| Header | Value | When |
|---|---|---|
| `X-RateLimit-Limit` | Total requests allowed in window | Always |
| `X-RateLimit-Remaining` | Requests remaining this window | Always |
| `X-RateLimit-Reset` | Unix epoch when window resets | Always (where applicable) |
| `Retry-After` | Seconds until client may retry | 429 responses only |

```go
// pkg/middleware/rate_limit_headers.go
package middleware

import (
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
)

// SetRateLimitHeaders writes standard rate limit headers.
// resetAt: time when the current window expires (zero value omits X-RateLimit-Reset).
func SetRateLimitHeaders(c *gin.Context, limit, remaining int, resetAt time.Time) {
	c.Header("X-RateLimit-Limit", strconv.Itoa(limit))
	c.Header("X-RateLimit-Remaining", strconv.Itoa(remaining))
	if !resetAt.IsZero() {
		c.Header("X-RateLimit-Reset", strconv.FormatInt(resetAt.Unix(), 10))
	}
}

// AbortRateLimited sends a 429 with Retry-After and standard rate limit headers.
func AbortRateLimited(c *gin.Context, limit int, retryAfterSec int) {
	SetRateLimitHeaders(c, limit, 0, time.Now().Add(time.Duration(retryAfterSec)*time.Second))
	c.Header("Retry-After", strconv.Itoa(retryAfterSec))
	c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
		"error":       "rate limit exceeded",
		"retry_after": retryAfterSec,
	})
}
```

**Why always set headers:** Clients and API gateways use `X-RateLimit-Remaining` to implement proactive back-off. If headers only appear on 429, clients have no signal until they are already blocked.

---

## Graceful Degradation

When Redis is unavailable, fall back to an in-memory limiter instead of blocking all requests or panicking.

```go
// pkg/middleware/rate_limiter_fallback.go
package middleware

import (
	"context"
	"log/slog"
	"sync"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/redis/go-redis/v9"
	"golang.org/x/time/rate"
)

// FallbackRateLimiter tries Redis first; on error falls back to in-memory.
// This ensures availability over strict accuracy when Redis is down.
func FallbackRateLimiter(redisCfg RedisTokenBucketConfig, memCfg MemoryRateLimiterConfig, done <-chan struct{}) gin.HandlerFunc {
	redisLimiter := RedisTokenBucketLimiter(redisCfg)

	// In-memory fallback, initialized once.
	memLimiter := MemoryRateLimiter(memCfg, done)

	// Track Redis health to avoid per-request PING overhead.
	var (
		mu          sync.RWMutex
		redisHealthy = true
		lastCheck    time.Time
		checkEvery   = 10 * time.Second
	)

	checkRedis := func(ctx context.Context) bool {
		mu.RLock()
		healthy := redisHealthy
		since := time.Since(lastCheck)
		mu.RUnlock()

		if since < checkEvery {
			return healthy
		}

		mu.Lock()
		defer mu.Unlock()
		err := redisCfg.Client.Ping(ctx).Err()
		redisHealthy = err == nil
		lastCheck = time.Now()
		if err != nil {
			slog.Warn("redis unhealthy, using in-memory rate limiter", "err", err)
		} else if !healthy {
			slog.Info("redis recovered, resuming redis rate limiter")
		}
		return redisHealthy
	}

	return func(c *gin.Context) {
		if checkRedis(c.Request.Context()) {
			redisLimiter(c)
		} else {
			memLimiter(c)
		}
	}
}
```

**Why fail open (allow) on Redis errors:** Blocking all traffic because a cache is down is worse than briefly allowing excess traffic. Log the error, alert on it, but keep the API serving. If strict enforcement is required, fail closed — change `c.Next()` to `c.AbortWithStatus(503)` in the Redis error paths above.

**Registration example:**

```go
// main.go
done := make(chan struct{})
defer close(done)

r.Use(middleware.FallbackRateLimiter(
    middleware.RedisTokenBucketConfig{
        Client:    rdb,
        Capacity:  100,
        Rate:      10,
        KeyPrefix: "rl:api:",
    },
    middleware.MemoryRateLimiterConfig{
        Rate:  10,
        Burst: 20,
    },
    done,
))
```
