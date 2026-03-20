# Logging

← [Back to README](../README.md)

## Approved Library

RBKL uses **`log/slog`** (Go 1.21+ standard library) for all structured logging. No third-party logging library is permitted without a Platform Engineering approval.

> **Why `slog`?** It is part of the Go standard library (no dependency), supports structured key-value attributes natively, is fast, and integrates with OpenTelemetry trace IDs.

---

## Setup

### Logger Initialization

Initialize the logger once in `main.go` and pass it through dependency injection. Do **not** use a global logger variable.

```go
package main

import (
    "log/slog"
    "os"

    "github.com/rbkl/my-service/internal/config"
)

func newLogger(cfg *config.Config) *slog.Logger {
    var handler slog.Handler

    opts := &slog.HandlerOptions{
        Level:     cfg.LogLevel, // slog.LevelDebug, LevelInfo, LevelWarn, LevelError
        AddSource: cfg.Env != "production",
    }

    if cfg.Env == "production" {
        // JSON handler for production — consumed by log aggregators
        handler = slog.NewJSONHandler(os.Stdout, opts)
    } else {
        // Text handler for local development — human-readable
        handler = slog.NewTextHandler(os.Stdout, opts)
    }

    return slog.New(handler)
}
```

### Default Log Levels by Environment

| Environment | Minimum Level |
|---|---|
| `local` | `DEBUG` |
| `dev` | `DEBUG` |
| `staging` | `INFO` |
| `production` | `INFO` |

---

## Log Levels

| Level | When to Use | Example |
|---|---|---|
| `DEBUG` | Verbose diagnostic information useful only during development | Decoded request body, cache hit/miss, SQL query |
| `INFO` | Normal operational events | Server started, request completed, user logged in |
| `WARN` | Something unexpected but recoverable occurred | Retried a failing dependency, deprecated API called |
| `ERROR` | An operation failed and requires attention | DB connection failed, unhandled error, data corruption |

**Never use `ERROR` for client errors (4xx).** A missing resource or bad request is not a server error — log it at `INFO` or `DEBUG`.

```go
// ✅ Client error — INFO
logger.InfoContext(ctx, "user not found", "user_id", id)

// ✅ Server error — ERROR
logger.ErrorContext(ctx, "database query failed", "error", err, "user_id", id)
```

---

## Structured Attributes

Always use key-value pairs, never string concatenation.

```go
// ✅ Good — structured, parseable
logger.InfoContext(ctx, "request completed",
    "method", r.Method,
    "path", r.URL.Path,
    "status", status,
    "duration_ms", duration.Milliseconds(),
    "request_id", middleware.GetReqID(ctx),
)

// ❌ Bad — unstructured
logger.Info(fmt.Sprintf("request %s %s completed in %dms", r.Method, r.URL.Path, duration.Milliseconds()))
```

### Standard Attribute Keys

Use these keys consistently across all services so log aggregation queries work uniformly:

| Key | Type | Description |
|---|---|---|
| `request_id` | string | Unique request identifier (from `X-Request-ID`) |
| `trace_id` | string | OpenTelemetry trace ID |
| `span_id` | string | OpenTelemetry span ID |
| `user_id` | string | Authenticated user ID |
| `service` | string | Service name (set at logger creation) |
| `env` | string | Deployment environment |
| `error` | error | Go error value |
| `duration_ms` | int64 | Duration in milliseconds |
| `method` | string | HTTP method |
| `path` | string | HTTP path (no query string) |
| `status` | int | HTTP response status code |

---

## Context Propagation

Pass the logger through `context.Context` so it can carry correlation IDs automatically.

```go
// Inject logger with base attributes into context
func WithLogger(ctx context.Context, logger *slog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey{}, logger)
}

// Extract logger from context (falls back to default if not set)
func FromContext(ctx context.Context) *slog.Logger {
    if l, ok := ctx.Value(loggerKey{}).(*slog.Logger); ok && l != nil {
        return l
    }
    return slog.Default()
}
```

In request middleware, inject the logger enriched with the request ID:

```go
func RequestLogger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        reqID := middleware.GetReqID(r.Context())
        logger := slog.Default().With(
            "request_id", reqID,
            "method", r.Method,
            "path", r.URL.Path,
        )
        ctx := log.WithLogger(r.Context(), logger)
        ww := middleware.NewWrapResponseWriter(w, r.ProtoMajor)
        start := time.Now()

        next.ServeHTTP(ww, r.WithContext(ctx))

        logger.InfoContext(ctx, "request completed",
            "status", ww.Status(),
            "bytes", ww.BytesWritten(),
            "duration_ms", time.Since(start).Milliseconds(),
        )
    })
}
```

---

## Sensitive Data Redaction

**Never log the following data**, even at `DEBUG` level:

- Passwords, secrets, API keys, tokens
- Full credit card numbers or PAN data
- Social Security Numbers or national IDs
- Private keys or certificates
- Full OAuth `code` or `refresh_token` values

Use a `Redacted` type to prevent accidental logging:

```go
type Redacted string

func (r Redacted) LogValue() slog.Value {
    return slog.StringValue("[REDACTED]")
}

// Usage
logger.Info("auth attempt", "username", username, "password", Redacted(password))
// Output: auth attempt username=alice password=[REDACTED]
```

---

## OpenTelemetry Integration

When tracing is enabled, enrich logs with trace and span IDs from the context:

```go
import "go.opentelemetry.io/otel/trace"

func enrichWithTrace(ctx context.Context, logger *slog.Logger) *slog.Logger {
    span := trace.SpanFromContext(ctx)
    if !span.IsRecording() {
        return logger
    }
    sc := span.SpanContext()
    return logger.With(
        "trace_id", sc.TraceID().String(),
        "span_id", sc.SpanID().String(),
    )
}
```

---

## Log Output Format

### Production (JSON)
```json
{
  "time": "2026-03-20T12:34:56.789Z",
  "level": "INFO",
  "msg": "request completed",
  "service": "user-service",
  "env": "production",
  "request_id": "abc-123",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "user_id": "usr_01HX...",
  "method": "GET",
  "path": "/v1/users/usr_01HX",
  "status": 200,
  "duration_ms": 42
}
```

### Development (Text)
```
time=2026-03-20T12:34:56.789Z level=INFO msg="request completed" service=user-service request_id=abc-123 method=GET path=/v1/users/usr_01HX status=200 duration_ms=42
```

---

## Prohibited Patterns

```go
// ❌ Never use fmt.Println or fmt.Printf for logging
fmt.Println("something happened")

// ❌ Never use the `log` package (log.Print, log.Fatal, etc.)
log.Printf("error: %v", err)

// ❌ Never use log.Fatal outside of main()
log.Fatal("startup failed")    // kills the process without cleanup

// ❌ Never swallow errors silently
if err != nil { return } // Where is the log?

// ❌ Never use string formatting for attributes
logger.Info(fmt.Sprintf("user %s logged in", userID))
```

---

_[← Web Servers](web-servers.md) · [Authentication →](authentication.md)_
