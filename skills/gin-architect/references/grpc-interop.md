# gRPC Interoperability Reference

How to run Gin HTTP and gRPC in the same Go project. Covers when to add gRPC, service-to-service communication, shared business logic between HTTP and gRPC handlers, and the multiplexer pattern.

**Default stance:** Do not add gRPC unless you have a real need. REST with Gin is sufficient for most projects. gRPC adds protobuf compilation, code generation, and operational complexity.

**Cost: HIGH (4/5).** Only justified when REST genuinely cannot meet requirements.

---

## Table of Contents

1. [When to Use gRPC](#1-when-to-use-grpc)
2. [Project Structure](#2-project-structure)
3. [Shared Service Layer](#3-shared-service-layer)
4. [gRPC Server Setup](#4-grpc-server-setup)
5. [HTTP + gRPC Multiplexer (cmux)](#5-http--grpc-multiplexer-cmux)
6. [gRPC Client for Service-to-Service](#6-grpc-client-for-service-to-service)
7. [gRPC-Gateway (REST Proxy)](#7-grpc-gateway-rest-proxy)
8. [Docker Compose with gRPC](#8-docker-compose-with-grpc)
9. [Cross-Skill References](#9-cross-skill-references)

---

## 1. When to Use gRPC

```
START: Do you need gRPC?
  ├── All clients are web/mobile? → No. REST with Gin. Stop.
  └── Service-to-service communication?
      ├── < 5 services, simple payloads? → REST is fine. Stop.
      └── High throughput, strict contracts, streaming needed?
          ├── Streaming (bidirectional)? → gRPC. Worth the complexity.
          └── Just strict contracts + perf? → gRPC, but consider
                gRPC-Gateway for external REST consumers.
```

### Indicators you DO need gRPC

- Bidirectional or server-side streaming (e.g., real-time feeds, long-lived connections)
- Tight latency budgets between internal services (< 10ms p99)
- Strict schema contracts enforced at build time (protobuf compilation fails on mismatch)
- 10+ internal services communicating with each other

### Indicators you do NOT need gRPC

- All consumers are browsers or mobile apps (REST is simpler)
- Fewer than 5 services with low call frequency
- Team unfamiliar with protobuf toolchain
- No existing gRPC infrastructure in your org

---

## 2. Project Structure

```
myapp/
├── cmd/
│   ├── api/
│   │   └── main.go          # HTTP entry point (Gin)
│   └── server/
│       └── main.go          # Combined HTTP + gRPC entry point
├── internal/
│   └── user/
│       ├── domain/
│       │   └── user.go      # Domain types (shared)
│       ├── usecase/
│       │   └── user_service.go  # Business logic (shared by both handlers)
│       ├── handler/
│       │   └── user_handler.go  # Gin HTTP handlers
│       ├── grpchandler/
│       │   └── user_handler.go  # gRPC handlers (calls same usecase)
│       └── repository/
│           └── user_repo.go
├── proto/
│   ├── buf.yaml             # buf configuration
│   ├── buf.gen.yaml         # code generation config
│   └── user/v1/
│       └── user.proto
├── gen/                     # Generated protobuf Go code (do not edit)
│   └── user/v1/
│       ├── user.pb.go
│       └── user_grpc.pb.go
└── go.mod
```

Key principle: both `handler/` (HTTP) and `grpchandler/` (gRPC) import from `usecase/`. Business logic lives once.

### buf.yaml (protobuf management)

```yaml
version: v2
modules:
  - path: proto
deps:
  - buf.build/googleapis/googleapis
```

### buf.gen.yaml

```yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt:
      - paths=source_relative
  - remote: buf.build/grpc/go
    out: gen
    opt:
      - paths=source_relative
```

Generate with: `buf generate`

---

## 3. Shared Service Layer

Both handlers call the same `UserService`. This is the most important pattern — business logic is never duplicated.

```go
// internal/user/usecase/user_service.go
package usecase

import (
    "context"
    "log/slog"

    "myapp/internal/user/domain"
)

type UserService struct {
    repo UserRepository
    log  *slog.Logger
}

func NewUserService(repo UserRepository, log *slog.Logger) *UserService {
    return &UserService{repo: repo, log: log}
}

func (s *UserService) GetByID(ctx context.Context, id string) (*domain.User, error) {
    return s.repo.FindByID(ctx, id)
}
```

```go
// internal/user/handler/user_handler.go — HTTP (Gin)
package handler

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "myapp/internal/user/usecase"
)

type UserHandler struct {
    svc *usecase.UserService
}

func NewUserHandler(svc *usecase.UserService) *UserHandler {
    return &UserHandler{svc: svc}
}

func (h *UserHandler) GetByID(c *gin.Context) {
    id := c.Param("id")
    user, err := h.svc.GetByID(c.Request.Context(), id)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
        return
    }
    c.JSON(http.StatusOK, gin.H{"id": user.ID, "name": user.Name})
}
```

```go
// internal/user/grpchandler/user_handler.go — gRPC
package grpchandler

import (
    "context"

    pb "myapp/gen/user/v1"
    "myapp/internal/user/usecase"
)

type UserGRPCHandler struct {
    pb.UnimplementedUserServiceServer
    svc *usecase.UserService
}

func NewUserGRPCHandler(svc *usecase.UserService) *UserGRPCHandler {
    return &UserGRPCHandler{svc: svc}
}

func (h *UserGRPCHandler) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    user, err := h.svc.GetByID(ctx, req.GetId())
    if err != nil {
        return nil, err // wrap with grpc/status codes in production
    }
    return &pb.GetUserResponse{Id: user.ID, Name: user.Name}, nil
}
```

---

## 4. gRPC Server Setup

```go
// cmd/server/main.go
package main

import (
    "log/slog"
    "net"
    "os"

    "google.golang.org/grpc"

    pb "myapp/gen/user/v1"
    "myapp/internal/user/grpchandler"
)

func startGRPC(handler *grpchandler.UserGRPCHandler) {
    log := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Error("failed to listen", "error", err)
        os.Exit(1)
    }

    grpcServer := grpc.NewServer(
        grpc.ChainUnaryInterceptor(loggingInterceptor(log)),
    )
    pb.RegisterUserServiceServer(grpcServer, handler)

    log.Info("gRPC server starting", "addr", ":50051")
    if err := grpcServer.Serve(lis); err != nil {
        log.Error("gRPC server failed", "error", err)
    }
}

func loggingInterceptor(log *slog.Logger) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
        resp, err := handler(ctx, req)
        log.Info("grpc call", "method", info.FullMethod, "error", err)
        return resp, err
    }
}
```

---

## 5. HTTP + gRPC Multiplexer (cmux)

Run HTTP and gRPC on the same port using `cmux`. Useful when firewall rules restrict port exposure.

```go
import (
    "net"
    "net/http"

    "github.com/soheilhy/cmux"
    "google.golang.org/grpc"
)

func runCombinedServer(ginEngine *gin.Engine, grpcServer *grpc.Server) error {
    lis, err := net.Listen("tcp", ":8080")
    if err != nil {
        return err
    }

    m := cmux.New(lis)

    // gRPC uses HTTP/2 with content-type: application/grpc
    grpcL := m.MatchWithWriters(
        cmux.HTTP2MatchHeaderFieldSendSettings("content-type", "application/grpc"),
    )
    httpL := m.Match(cmux.Any())

    go grpcServer.Serve(grpcL)
    go http.Serve(httpL, ginEngine) //nolint:gosec

    return m.Serve()
}
```

`go.mod` dependency: `github.com/soheilhy/cmux`

When to use cmux vs separate ports:
- Use cmux: single port required (load balancer, cloud firewall limitation)
- Use separate ports: simpler setup when you control infrastructure (preferred)

---

## 6. gRPC Client for Service-to-Service

```go
// internal/order/client/user_client.go
package client

import (
    "context"
    "log/slog"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    pb "myapp/gen/user/v1"
)

type UserClient struct {
    client pb.UserServiceClient
    log    *slog.Logger
}

func NewUserClient(addr string, log *slog.Logger) (*UserClient, error) {
    conn, err := grpc.NewClient(addr,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        return nil, err
    }
    return &UserClient{client: pb.NewUserServiceClient(conn), log: log}, nil
}

func (c *UserClient) GetUser(ctx context.Context, id string) (*pb.GetUserResponse, error) {
    return c.client.GetUser(ctx, &pb.GetUserRequest{Id: id})
}
```

Notes:
- Use `insecure.NewCredentials()` only in local/dev. Use TLS in production.
- `grpc.NewClient` (preferred over deprecated `grpc.Dial`)
- Reuse connections — create the client once and inject via DI
- `context.Context` from the calling Gin handler propagates naturally into gRPC calls (deadline/cancellation)

---

## 7. gRPC-Gateway (REST Proxy)

gRPC-Gateway generates an HTTP/JSON reverse proxy from protobuf annotations. Use when you need both gRPC (internal services) and REST (external clients) from a single `.proto` definition.

### proto annotation example

```protobuf
syntax = "proto3";

import "google/api/annotations.proto";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {
    option (google.api.http) = {
      get: "/v1/users/{id}"
    };
  }
}
```

### gateway setup

```go
import (
    "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    pb "myapp/gen/user/v1"
)

func newGateway(ctx context.Context, grpcAddr string) (http.Handler, error) {
    mux := runtime.NewServeMux()
    opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
    err := pb.RegisterUserServiceHandlerFromEndpoint(ctx, mux, grpcAddr, opts)
    return mux, err
}
```

When to use gRPC-Gateway:
- Internal gRPC services that must also expose a public REST API
- Migration path: existing REST clients, new gRPC-first internal architecture
- Avoid if you only need one protocol — it adds another layer to debug

---

## 8. Docker Compose with gRPC

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8080:8080"    # HTTP (Gin)
      - "50051:50051"  # gRPC
    environment:
      - APP_ENV=development
    depends_on:
      - db

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
```

Multi-stage Dockerfile with buf generation:

```dockerfile
# Stage 1: generate protobuf
FROM bufbuild/buf:latest AS proto-gen
WORKDIR /workspace
COPY proto/ proto/
COPY buf.yaml buf.gen.yaml ./
RUN buf generate

# Stage 2: build Go binary
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY --from=proto-gen /workspace/gen ./gen
COPY . .
RUN go build -o server ./cmd/server

# Stage 3: runtime
FROM alpine:3.21
COPY --from=builder /app/server /server
EXPOSE 8080 50051
CMD ["/server"]
```

---

## 9. Cross-Skill References

| Topic | Reference |
|-------|-----------|
| Clean architecture layering (usecase/handler separation) | `references/clean-architecture.md` |
| Service decomposition decisions | `references/system-design.md` |
| Deployment, ports, health checks | `gin-deploy` skill |
| Gin middleware patterns | `references/middleware-patterns.md` |

---

## Quick Reference

| Item | Value |
|------|-------|
| Official gRPC package | `google.golang.org/grpc` |
| Protobuf management | `buf` (not raw `protoc`) |
| Logging | `log/slog` (structured, JSON) |
| Context propagation | `context.Context` — gRPC passes it natively |
| Multiplexer | `github.com/soheilhy/cmux` |
| REST proxy | `github.com/grpc-ecosystem/grpc-gateway/v2` |
| Generated code dir | `gen/` — never edit manually |
| Complexity cost | HIGH (4/5) — justify before adding |
