# Object Storage Reference

Practical guide to integrating S3-compatible object storage in Go Gin APIs. Covers MinIO for local dev,
AWS SDK v2, upload/download patterns, presigned URLs, and multipart uploads.

---

## Table of Contents

1. [When to Use Object Storage](#1-when-to-use-object-storage)
2. [S3 Client Setup](#2-s3-client-setup)
3. [Upload Patterns](#3-upload-patterns)
4. [Download and Streaming](#4-download-and-streaming)
5. [Presigned URLs](#5-presigned-urls)
6. [Multipart Upload for Large Files](#6-multipart-upload-for-large-files)
7. [MinIO Docker Compose for Dev](#7-minio-docker-compose-for-dev)
8. [Cross-Skill References](#8-cross-skill-references)

---

## 1. When to Use Object Storage

```
START: Are you storing files > 1MB or binary blobs?
  ├── No  → PostgreSQL BYTEA or local filesystem. Done.
  └── Yes → Do you need CDN delivery?
      ├── Yes → S3 + CloudFront / CDN. Done.
      └── No  → Is it user-uploaded content?
          ├── Yes → S3 with presigned PUT URLs (client uploads directly, server never touches bytes)
          └── No  → S3 with server-side upload (internal processing pipelines, etc.)
```

**Cost gate:** S3 charges per request + per GB stored + egress. For internal files rarely downloaded,
PostgreSQL BYTEA (< 5 MB) or a mounted volume is simpler and cheaper. Choose object storage when you
expect > 100 MB total data or need CDN/direct-client access.

---

## 2. S3 Client Setup

Dependencies:

```
go get github.com/aws/aws-sdk-go-v2
go get github.com/aws/aws-sdk-go-v2/config
go get github.com/aws/aws-sdk-go-v2/credentials
go get github.com/aws/aws-sdk-go-v2/service/s3
```

```go
// pkg/storage/s3.go
package storage

import (
	"context"
	"fmt"
	"log/slog"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/credentials"
	"github.com/aws/aws-sdk-go-v2/service/s3"
)

// S3Client wraps the AWS SDK client with bucket context.
// Compatible with AWS S3, MinIO, and Cloudflare R2.
type S3Client struct {
	client *s3.Client
	bucket string
	logger *slog.Logger
}

// NewS3Client creates a client for any S3-compatible backend.
// Pass empty endpoint for real AWS (uses regional endpoint resolution).
func NewS3Client(endpoint, region, accessKey, secretKey, bucket string) (*S3Client, error) {
	optFns := []func(*config.LoadOptions) error{
		config.WithRegion(region),
		config.WithCredentialsProvider(
			credentials.NewStaticCredentialsProvider(accessKey, secretKey, ""),
		),
	}

	cfg, err := config.LoadDefaultConfig(context.Background(), optFns...)
	if err != nil {
		return nil, fmt.Errorf("storage: load config: %w", err)
	}

	clientOpts := []func(*s3.Options){}
	if endpoint != "" {
		// MinIO / R2 / custom S3-compatible endpoint
		clientOpts = append(clientOpts, func(o *s3.Options) {
			o.BaseEndpoint = aws.String(endpoint)
			o.UsePathStyle = true // required for MinIO
		})
	}

	return &S3Client{
		client: s3.NewFromConfig(cfg, clientOpts...),
		bucket: bucket,
		logger: slog.Default().With("component", "s3"),
	}, nil
}
```

**Environment variables** (map to constructor args via config struct):

```
S3_ENDPOINT=http://localhost:9000   # empty for real AWS
S3_REGION=us-east-1
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=myapp
```

---

## 3. Upload Patterns

### Thin Handler (multipart form)

```go
// internal/handler/file_handler.go
package handler

import (
	"net/http"

	"github.com/gin-gonic/gin"
	"yourmodule/internal/service"
)

type FileHandler struct {
	svc service.FileService
}

const maxUploadSize = 100 << 20 // 100 MB

func (h *FileHandler) Upload(c *gin.Context) {
	// Limit body before parsing to prevent OOM
	c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, maxUploadSize)

	file, header, err := c.Request.FormFile("file")
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid file: " + err.Error()})
		return
	}
	defer file.Close()

	contentType := header.Header.Get("Content-Type")
	if contentType == "" {
		contentType = "application/octet-stream"
	}

	url, err := h.svc.Upload(c.Request.Context(), file, header.Filename, contentType)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "upload failed"})
		return
	}

	c.JSON(http.StatusCreated, gin.H{"url": url})
}
```

### Service Layer

```go
// internal/service/file_service.go
package service

import (
	"context"
	"fmt"
	"io"
	"path/filepath"

	"github.com/google/uuid"
	"yourmodule/pkg/storage"
)

type FileService interface {
	Upload(ctx context.Context, r io.Reader, filename, contentType string) (string, error)
}

type fileService struct {
	store *storage.S3Client
}

func (s *fileService) Upload(ctx context.Context, r io.Reader, filename, contentType string) (string, error) {
	key := fmt.Sprintf("uploads/%s%s", uuid.New().String(), filepath.Ext(filename))

	url, err := s.store.PutObject(ctx, key, contentType, r)
	if err != nil {
		return "", fmt.Errorf("file service upload: %w", err)
	}
	return url, nil
}
```

### Storage Method

```go
// pkg/storage/s3.go (continued)
import (
	"github.com/aws/aws-sdk-go-v2/service/s3"
	"github.com/aws/aws-sdk-go-v2/aws"
)

// PutObject uploads a reader's content and returns the object URL.
func (c *S3Client) PutObject(ctx context.Context, key, contentType string, body io.Reader) (string, error) {
	_, err := c.client.PutObject(ctx, &s3.PutObjectInput{
		Bucket:      aws.String(c.bucket),
		Key:         aws.String(key),
		Body:        body,
		ContentType: aws.String(contentType),
	})
	if err != nil {
		return "", fmt.Errorf("s3 put object %q: %w", key, err)
	}

	c.logger.InfoContext(ctx, "object uploaded", "key", key)
	return fmt.Sprintf("s3://%s/%s", c.bucket, key), nil
}
```

---

## 4. Download and Streaming

Stream S3 objects directly to the Gin response writer — avoid loading into memory.

```go
// pkg/storage/s3.go (continued)

// GetObject streams an S3 object to the provided writer.
func (c *S3Client) GetObject(ctx context.Context, key string, w io.Writer) (string, error) {
	result, err := c.client.GetObject(ctx, &s3.GetObjectInput{
		Bucket: aws.String(c.bucket),
		Key:    aws.String(key),
	})
	if err != nil {
		return "", fmt.Errorf("s3 get object %q: %w", key, err)
	}
	defer result.Body.Close()

	contentType := aws.ToString(result.ContentType)
	if _, err := io.Copy(w, result.Body); err != nil {
		return "", fmt.Errorf("s3 stream object %q: %w", key, err)
	}
	return contentType, nil
}
```

```go
// internal/handler/file_handler.go

func (h *FileHandler) Download(c *gin.Context) {
	key := c.Param("key")

	contentType, err := h.svc.Download(c.Request.Context(), key, c.Writer)
	if err != nil {
		c.JSON(http.StatusNotFound, gin.H{"error": "object not found"})
		return
	}

	c.Header("Content-Type", contentType)
	c.Header("Content-Disposition", "attachment; filename="+filepath.Base(key))
	// Body already written by GetObject streaming above
}
```

---

## 5. Presigned URLs

Presigned URLs delegate upload/download directly between client and S3 — your server issues a
time-limited signed URL and never proxies the bytes.

```go
// pkg/storage/s3.go (continued)
import "github.com/aws/aws-sdk-go-v2/service/s3/presigned"

// PresignPutURL generates a signed URL for direct client upload.
// expiry: typically 15 minutes for uploads, 1 hour for downloads.
func (c *S3Client) PresignPutURL(ctx context.Context, key, contentType string, expiry time.Duration) (string, error) {
	presigner := s3.NewPresignClient(c.client)

	req, err := presigner.PresignPutObject(ctx, &s3.PutObjectInput{
		Bucket:      aws.String(c.bucket),
		Key:         aws.String(key),
		ContentType: aws.String(contentType),
	}, s3.WithPresignExpires(expiry))
	if err != nil {
		return "", fmt.Errorf("presign put %q: %w", key, err)
	}
	return req.URL, nil
}

// PresignGetURL generates a signed URL for direct client download.
func (c *S3Client) PresignGetURL(ctx context.Context, key string, expiry time.Duration) (string, error) {
	presigner := s3.NewPresignClient(c.client)

	req, err := presigner.PresignGetObject(ctx, &s3.GetObjectInput{
		Bucket: aws.String(c.bucket),
		Key:    aws.String(key),
	}, s3.WithPresignExpires(expiry))
	if err != nil {
		return "", fmt.Errorf("presign get %q: %w", key, err)
	}
	return req.URL, nil
}
```

**Handler — request presigned upload URL:**

```go
func (h *FileHandler) PresignUpload(c *gin.Context) {
	var req struct {
		Filename    string `json:"filename" binding:"required"`
		ContentType string `json:"content_type" binding:"required"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	key := fmt.Sprintf("uploads/%s%s", uuid.New().String(), filepath.Ext(req.Filename))
	url, err := h.store.PresignPutURL(c.Request.Context(), key, req.ContentType, 15*time.Minute)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "could not generate URL"})
		return
	}

	c.JSON(http.StatusOK, gin.H{
		"upload_url": url,
		"key":        key,
		"expires_in": 900, // seconds
	})
}
```

Client then `PUT`s directly to `upload_url` with the file bytes and matching `Content-Type` header.

---

## 6. Multipart Upload for Large Files

For files > 100 MB, use S3 multipart upload to avoid timeouts and enable resumable transfers.

```go
// pkg/storage/multipart.go
package storage

import (
	"context"
	"fmt"
	"io"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/feature/s3/manager"
	"github.com/aws/aws-sdk-go-v2/service/s3"
)

const partSize = 10 << 20 // 10 MB per part (minimum 5 MB required by S3)

// PutLargeObject uses the SDK upload manager (handles multipart automatically).
func (c *S3Client) PutLargeObject(ctx context.Context, key, contentType string, body io.Reader) error {
	uploader := manager.NewUploader(c.client, func(u *manager.Uploader) {
		u.PartSize = partSize
		u.Concurrency = 3 // parallel part uploads
	})

	_, err := uploader.Upload(ctx, &s3.PutObjectInput{
		Bucket:      aws.String(c.bucket),
		Key:         aws.String(key),
		Body:        body,
		ContentType: aws.String(contentType),
	})
	if err != nil {
		return fmt.Errorf("s3 multipart upload %q: %w", key, err)
	}

	c.logger.InfoContext(ctx, "large object uploaded", "key", key)
	return nil
}
```

`manager.NewUploader` transparently falls back to a single-part upload when body < 5 MB, so this
method is safe to use for all sizes.

---

## 7. MinIO Docker Compose for Dev

```yaml
# docker-compose.yml (dev only)
services:
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"   # S3 API — point S3_ENDPOINT=http://localhost:9000
      - "9001:9001"   # Web console — open http://localhost:9001
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 5s
      timeout: 5s
      retries: 5

  # One-shot container that creates the default bucket on first boot
  createbuckets:
    image: minio/mc:latest
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
        mc alias set local http://minio:9000 minioadmin minioadmin;
        mc mb --ignore-existing local/myapp;
        mc anonymous set download local/myapp/public;
        exit 0;
      "

volumes:
  minio_data:
```

**Matching `.env.dev`:**

```
S3_ENDPOINT=http://localhost:9000
S3_REGION=us-east-1
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=myapp
```

MinIO is wire-compatible with AWS S3 SDK v2. Switch to real AWS by removing `S3_ENDPOINT` and
setting real credentials — no code changes required.

---

## 8. Cross-Skill References

| Topic | Skill | File |
|---|---|---|
| Route registration for upload/download endpoints | `gin-api` | `references/routing.md` |
| Middleware for auth-gating upload endpoints | `gin-api` | `references/middleware.md` |
| Storing S3 keys in PostgreSQL (file metadata table) | `gin-architect` | `references/database-patterns.md` |
| Env var injection for `S3_*` in production | `gin-deploy` | `references/environment-config.md` |
| Docker Compose composition for full stack | `gin-deploy` | `references/docker-compose.md` |
