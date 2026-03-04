# Dockerfile Reference

This file covers Docker packaging for Go Gin APIs: multi-stage build rationale, base image comparison (distroless vs Alpine vs scratch), build arguments and secrets, layer caching, non-root user, HEALTHCHECK instruction, image size optimization, and a complete production Dockerfile. Use when you need to understand the tradeoffs or customize the build.

> **Architectural recommendation:** Docker patterns are not part of the Gin framework. These are mainstream Go community patterns.

## Table of Contents

1. [Multi-Stage Build Explained](#multi-stage-build-explained)
2. [Base Image Comparison](#base-image-comparison)
3. [Layer Caching Optimization](#layer-caching-optimization)
4. [Build Arguments and Secrets](#build-arguments-and-secrets)
5. [Non-Root User](#non-root-user)
6. [HEALTHCHECK Instruction](#healthcheck-instruction)
7. [Image Size Optimization](#image-size-optimization)
8. [.dockerignore](#dockerignore)
9. [Complete Production Dockerfile](#complete-production-dockerfile)

---

## Multi-Stage Build Explained

A multi-stage build uses multiple `FROM` instructions in one Dockerfile. Each stage is independent — only explicitly copied artifacts carry forward.

**Why it matters for Go:** The Go toolchain (~800 MB) is only needed at compile time. The final image only needs the compiled binary (~10 MB).

```
Stage 1 (builder)          Stage 2 (runtime)
─────────────────          ─────────────────
golang:1.24 (~800 MB)      distroless (~2 MB)
+ source code              + /server binary (~8 MB)
+ go modules               ────────────────────────
+ compiled binary          Final image: ~10 MB
```

The key directive:

```dockerfile
COPY --from=builder /app/server /server
```

Only the compiled binary crosses stage boundaries — source code, module cache, and the Go toolchain stay in the builder stage and are discarded.

---

## Base Image Comparison

| Image | Size | Shell | Package manager | Use case |
|-------|------|-------|----------------|----------|
| `gcr.io/distroless/static-debian12` | ~2 MB | No | No | Production (recommended) |
| `gcr.io/distroless/base-debian12` | ~20 MB | No | No | When CGO is required |
| `alpine:3.21` | ~7 MB | sh | apk | When shell access needed |
| `scratch` | 0 MB | No | No | Smallest possible; no TLS certs |

**Distroless (recommended):**

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot
```

- No shell = no shell injection attacks
- Includes CA certificates and timezone data (unlike `scratch`)
- `nonroot` tag runs as UID 65532 by default — no `USER` directive needed
- `CGO_ENABLED=0` required in builder stage

**When to use Alpine instead:**

```dockerfile
FROM alpine:3.21

# Alpine needs ca-certificates for HTTPS outbound calls if not using distroless
RUN apk add --no-cache ca-certificates tzdata
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=builder /app/server /app/server
ENTRYPOINT ["/app/server"]
```

Use Alpine when you need: shell for debugging, `apk` to install runtime deps (e.g., `git`, `openssh-client`), or when the team is unfamiliar with distroless.

**When to use scratch:**

```dockerfile
FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

Use scratch only when you need the absolute minimum image size and can manage TLS certificates manually. No timezone data — `time.LoadLocation` will fail without explicit tz data copy.

---

## Layer Caching Optimization

Docker caches each layer. A cache miss invalidates all subsequent layers. Ordering matters.

**Anti-pattern — cache broken by every code change:**

```dockerfile
# BAD: source copy before go mod download
COPY . .
RUN go mod download  # re-runs on every source change
RUN go build ...
```

**Correct pattern — dependencies cached separately:**

```dockerfile
# GOOD: module files copied first
COPY go.mod go.sum ./
RUN go mod download  # only re-runs when go.mod or go.sum changes

COPY . .             # source changes don't invalidate module cache
RUN go build ...
```

**Further optimization with BuildKit cache mount:**

```dockerfile
# syntax=docker/dockerfile:1

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o /app/server ./cmd/api
```

The `--mount=type=cache` keeps the module and build caches across builds on the same host, making rebuilds ~5-10x faster when only source files change.

---

## Build Arguments and Secrets

**Build arguments (non-sensitive, embedded in image):**

```dockerfile
ARG VERSION=dev
ARG BUILD_TIME

RUN CGO_ENABLED=0 go build \
    -ldflags="-s -w -X main.Version=${VERSION} -X main.BuildTime=${BUILD_TIME}" \
    -o /app/server \
    ./cmd/api
```

Pass at build time:

```bash
docker build \
  --build-arg VERSION=$(git describe --tags --always) \
  --build-arg BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t myapp:latest .
```

Expose in the health endpoint:

```go
// Set via -ldflags at build time
var (
    Version   = "dev"
    BuildTime = "unknown"
)

func (h *HealthHandler) Check(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{
        "status":     "ok",
        "version":    Version,
        "build_time": BuildTime,
    })
}
```

**Secrets (never embedded — use BuildKit secret mount):**

```dockerfile
# syntax=docker/dockerfile:1

# Mount a secret during build without writing it to any layer
RUN --mount=type=secret,id=github_token \
    GITHUB_TOKEN=$(cat /run/secrets/github_token) \
    GONOSUMCHECK=github.com/myorg/* \
    GOFLAGS=-mod=mod \
    go mod download
```

Pass the secret at build time:

```bash
docker build --secret id=github_token,src=~/.github_token -t myapp .
```

**Warning:** Never use `ARG` for secrets — build args are visible in `docker history` and image metadata. Use BuildKit secret mounts for any sensitive value needed at build time.

---

## Non-Root User

Running as root inside a container is a security risk — a container escape would give root on the host. Always run as non-root.

**With distroless:nonroot — nothing extra needed:**

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot
# Already configured to run as UID 65532 (nonroot)
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

**With Alpine — create a dedicated user:**

```dockerfile
FROM alpine:3.21

RUN apk add --no-cache ca-certificates tzdata && \
    addgroup -S appgroup && \
    adduser -S -G appgroup -u 10001 appuser

COPY --from=builder /app/server /app/server
RUN chown appuser:appgroup /app/server

USER appuser
ENTRYPOINT ["/app/server"]
```

**File permissions:** If the app needs to write files (e.g., temp uploads), ensure the target directory is owned by the app user or use a volume with appropriate permissions.

---

## HEALTHCHECK Instruction

The `HEALTHCHECK` Dockerfile instruction tells Docker how to test whether the container is healthy. Docker Engine (and docker-compose) uses this — Kubernetes uses its own probe configuration instead.

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD ["/server", "-health-check"]
```

**Problem:** distroless has no shell and no `curl`/`wget`. Use a dedicated health-check binary or a flag in the main binary.

**Recommended pattern — embed health check in the binary:**

```go
// cmd/api/main.go
import (
    "flag"
    "fmt"
    "net/http"
    "os"
)

func main() {
    healthCheck := flag.Bool("health-check", false, "run health check and exit")
    flag.Parse()

    if *healthCheck {
        url := fmt.Sprintf("http://localhost:%s/health", os.Getenv("PORT"))
        resp, err := http.Get(url)
        if err != nil {
            os.Exit(1)
        }
        defer resp.Body.Close()
        if resp.StatusCode != http.StatusOK {
            os.Exit(1)
        }
        os.Exit(0)
    }

    // ... normal server startup
}
```

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD ["/server", "-health-check"]
```

**Kubernetes note:** When deploying to K8s, the `HEALTHCHECK` instruction is ignored. K8s uses `livenessProbe` and `readinessProbe` in the pod spec. See `references/kubernetes.md`.

---

## Image Size Optimization

Techniques to minimize the final image size:

**1. Use `-ldflags="-s -w"`**

```bash
go build -ldflags="-s -w" ...
```

- `-s` strips the symbol table (~30% size reduction)
- `-w` strips DWARF debug info (additional ~10% reduction)

**2. Use `-trimpath`**

```bash
go build -trimpath ...
```

Removes local file system paths from the binary. Improves reproducibility and hides build machine paths.

**3. Use `upx` compression (optional, tradeoff)**

```dockerfile
FROM builder AS compressor
RUN apt-get install -y upx-ucl && upx --best /app/server
```

UPX can reduce binary size by 50-70% but adds startup latency (~100-300ms) due to in-memory decompression. Use only for extremely size-constrained environments.

**4. Avoid copying unnecessary files**

Use `.dockerignore` to exclude test files, documentation, and development tooling from the build context.

**Size comparison for a typical Gin API binary:**

| Configuration | Binary size | Final image |
|---------------|-------------|-------------|
| Default build | ~12 MB | ~14 MB (distroless) |
| `-ldflags="-s -w"` | ~8 MB | ~10 MB |
| `-ldflags="-s -w"` + UPX | ~3 MB | ~5 MB |

---

## .dockerignore

```dockerignore
# Version control
.git
.gitignore
.github

# Development tools
.air.toml
air.toml
.golangci.yml
Makefile

# Local environment
.env
.env.local
.env.*
!.env.example

# Docker files (not needed in build context)
docker-compose*.yml
Dockerfile*

# Documentation
*.md
docs/
plans/

# Test artifacts
*_test.go
coverage.html
coverage.out
*.test

# Editor
.vscode/
.idea/
*.swp
```

**Critical:** Always exclude `.env` files from the build context. Even if you don't `COPY` them explicitly, they exist in the build context and could be accidentally included by a broad `COPY . .` in future changes.

---

## Complete Production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
# Build: docker build --build-arg VERSION=$(git describe --tags --always) -t myapp:latest .

# ── Stage 1: Dependency cache ─────────────────────────────────────────────────
FROM golang:1.24-bookworm AS deps

WORKDIR /build

# Copy only module files first — cached until go.mod/go.sum change
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download && go mod verify

# ── Stage 2: Builder ──────────────────────────────────────────────────────────
FROM deps AS builder

ARG VERSION=dev
ARG BUILD_TIME

# Copy source code
COPY . .

# Build statically linked binary
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-s -w -X main.Version=${VERSION} -X main.BuildTime=${BUILD_TIME}" \
    -trimpath \
    -o /app/server \
    ./cmd/api

# Verify the binary was built correctly
RUN /app/server -health-check 2>/dev/null || true

# ── Stage 3: Runtime ──────────────────────────────────────────────────────────
FROM gcr.io/distroless/static-debian12:nonroot

# Metadata labels
LABEL org.opencontainers.image.source="https://github.com/myorg/myapp"
LABEL org.opencontainers.image.description="Gin REST API"
LABEL org.opencontainers.image.licenses="MIT"

COPY --from=builder /app/server /server

# Expose port (documentation only — actual binding via -p or K8s service)
EXPOSE 8080

# Health check for Docker Engine and docker-compose
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD ["/server", "-health-check"]

# distroless:nonroot runs as UID 65532 — no USER directive needed
ENTRYPOINT ["/server"]
```

Build and run:

```bash
# Build
docker build \
  --build-arg VERSION=$(git describe --tags --always) \
  --build-arg BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t myapp:latest .

# Run locally
docker run --rm \
  -p 8080:8080 \
  -e DATABASE_URL="postgres://user:pass@host:5432/db?sslmode=require" \
  -e JWT_SECRET="your-secret" \
  -e GIN_MODE=release \
  myapp:latest

# Inspect image size
docker images myapp:latest
```
