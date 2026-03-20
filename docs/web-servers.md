# Web Servers & HTTP

← [Back to README](../README.md)

## Approved Stack

| Component | Library | Version |
|---|---|---|
| HTTP framework | `net/http` (stdlib) + `chi` router | `chi` v5 |
| Middleware | `chi/middleware` + custom RBKL middleware | — |
| JSON encoding | `encoding/json` (stdlib) or `json-iterator/go` | latest |

> **Why `chi`?** Chi is a lightweight, idiomatic router built on `net/http`. It adds zero magic — handlers are standard `http.HandlerFunc` values, so there is no vendor lock-in and testing is trivial. It also has an excellent middleware ecosystem.

---

## Router Setup

```go
package server

import (
    "net/http"
    "time"

    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
    rbklmw "github.com/rbkl/go-middleware"
)

func newRouter(cfg *config.Config, h *handler.Handler) http.Handler {
    r := chi.NewRouter()

    // ── Global middleware (applied to every request) ──────────────────────────
    r.Use(middleware.RequestID)          // Inject X-Request-ID header
    r.Use(rbklmw.RequestLogger)          // Structured request/response logging
    r.Use(middleware.Recoverer)          // Recover from panics, return 500
    r.Use(middleware.RealIP)             // Trust X-Forwarded-For / X-Real-IP
    r.Use(middleware.Timeout(30 * time.Second)) // Request deadline
    r.Use(rbklmw.CORS(cfg.AllowedOrigins))      // CORS policy

    // ── Public routes ─────────────────────────────────────────────────────────
    r.Get("/healthz", h.Health)
    r.Get("/readyz", h.Ready)

    // ── API v1 ────────────────────────────────────────────────────────────────
    r.Route("/v1", func(r chi.Router) {
        r.Use(rbklmw.Authenticate(cfg.JWKS))  // JWT authentication
        r.Use(rbklmw.Authorize)               // RBAC enforcement

        r.Route("/users", func(r chi.Router) {
            r.Get("/", h.ListUsers)
            r.Post("/", h.CreateUser)
            r.Route("/{userID}", func(r chi.Router) {
                r.Get("/", h.GetUser)
                r.Put("/", h.UpdateUser)
                r.Delete("/", h.DeleteUser)
            })
        })
    })

    return r
}
```

---

## Middleware Stack

Apply middleware **in this order** on every service. The ordering matters — each layer wraps the next.

| Order | Middleware | Purpose |
|---|---|---|
| 1 | `middleware.RequestID` | Assign unique ID to each request |
| 2 | `rbklmw.RequestLogger` | Log request start/end with correlation ID |
| 3 | `middleware.Recoverer` | Catch panics, return `500`, log the stack trace |
| 4 | `middleware.RealIP` | Extract real client IP from proxy headers |
| 5 | `middleware.Timeout` | Enforce a 30-second maximum request duration |
| 6 | `rbklmw.CORS` | Apply CORS policy from config |
| 7 | `rbklmw.Authenticate` | Validate JWT; populate claims in context |
| 8 | `rbklmw.Authorize` | Enforce RBAC roles against required permissions |

### Custom Middleware Pattern

All custom middleware must follow the standard `chi` / `net/http` pattern:

```go
func MyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Pre-processing
        ctx := context.WithValue(r.Context(), myKey{}, "value")
        // Call next handler
        next.ServeHTTP(w, r.WithContext(ctx))
        // Post-processing (if needed)
    })
}
```

---

## Request Handling

### Handler Structure

Handlers are thin: they decode the request, call a service, and encode the response. No business logic.

```go
package handler

type Handler struct {
    users  UserService
    logger *slog.Logger
}

func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    id := chi.URLParam(r, "userID")

    user, err := h.users.Get(ctx, id)
    if err != nil {
        h.writeError(w, r, err)
        return
    }

    h.writeJSON(w, http.StatusOK, user)
}
```

### JSON Helpers

Every service uses these two helper methods (or the shared `rbkl/go-middleware` equivalents):

```go
func (h *Handler) writeJSON(w http.ResponseWriter, status int, v any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if err := json.NewEncoder(w).Encode(v); err != nil {
        h.logger.Error("failed to encode response", "error", err)
    }
}

func (h *Handler) writeError(w http.ResponseWriter, r *http.Request, err error) {
    problem := toProblemDetails(r, err)
    h.writeJSON(w, problem.Status, problem)
}
```

See [Error Handling](error-handling.md) for the `toProblemDetails` implementation.

---

## Server Initialization & Graceful Shutdown

```go
package server

import (
    "context"
    "errors"
    "fmt"
    "log/slog"
    "net/http"
    "time"
)

type Server struct {
    http *http.Server
}

func New(cfg *config.Config, h *handler.Handler) *Server {
    return &Server{
        http: &http.Server{
            Addr:         fmt.Sprintf(":%d", cfg.Port),
            Handler:      newRouter(cfg, h),
            ReadTimeout:  10 * time.Second,
            WriteTimeout: 30 * time.Second,
            IdleTimeout:  120 * time.Second,
        },
    }
}

func (s *Server) Run(ctx context.Context) error {
    errCh := make(chan error, 1)

    go func() {
        slog.Info("server starting", "addr", s.http.Addr)
        if err := s.http.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
            errCh <- err
        }
        close(errCh)
    }()

    select {
    case err := <-errCh:
        return fmt.Errorf("server error: %w", err)
    case <-ctx.Done():
        slog.Info("shutdown signal received, draining connections")
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
        defer cancel()
        if err := s.http.Shutdown(shutdownCtx); err != nil {
            return fmt.Errorf("graceful shutdown failed: %w", err)
        }
        slog.Info("server stopped gracefully")
        return nil
    }
}
```

**Required timeouts:**

| Timeout | Value | Reason |
|---|---|---|
| `ReadTimeout` | 10s | Prevent slowloris attacks |
| `WriteTimeout` | 30s | Allow time for large responses |
| `IdleTimeout` | 120s | Keep-alive connection management |
| Request context timeout | 30s | Applied via middleware |
| Graceful shutdown | 15s | Drain in-flight requests |

---

## Health & Readiness Endpoints

Every service **must** expose these two endpoints (no authentication required):

### `GET /healthz` — Liveness

Returns `200 OK` if the process is alive. Used by Kubernetes to decide whether to restart the container.

```go
func (h *Handler) Health(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    _, _ = w.Write([]byte(`{"status":"ok"}`))
}
```

### `GET /readyz` — Readiness

Returns `200 OK` only if all dependencies (database, cache, etc.) are reachable. Used by Kubernetes to decide whether to send traffic.

```go
func (h *Handler) Ready(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    if err := h.db.PingContext(ctx); err != nil {
        h.logger.Warn("readiness check failed", "error", err)
        http.Error(w, `{"status":"unavailable"}`, http.StatusServiceUnavailable)
        return
    }

    w.WriteHeader(http.StatusOK)
    _, _ = w.Write([]byte(`{"status":"ready"}`))
}
```

---

## CORS Policy

CORS is configured via the `AllowedOrigins` config value. Default policy:

| Header | Value |
|---|---|
| `Access-Control-Allow-Origin` | Configurable (no wildcard in production) |
| `Access-Control-Allow-Methods` | `GET, POST, PUT, DELETE, OPTIONS` |
| `Access-Control-Allow-Headers` | `Authorization, Content-Type, X-Request-ID` |
| `Access-Control-Max-Age` | `300` |
| `Access-Control-Allow-Credentials` | `true` (only when origin is explicitly listed) |

**Never use `*` for `Access-Control-Allow-Origin` in production.**

---

_[← Project Structure](project-structure.md) · [Logging →](logging.md)_
