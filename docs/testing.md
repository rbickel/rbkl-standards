# Testing

← [Back to README](../README.md)

## Approved Libraries

| Library | Purpose | Version |
|---|---|---|
| `testing` | Test runner (stdlib) | — |
| `testify/assert` | Fluent assertions | v1 |
| `testify/require` | Fatal assertions | v1 |
| `testify/mock` | Mock objects | v1 |
| `mockery` | Mock code generation | v2 |
| `net/http/httptest` | HTTP handler testing (stdlib) | — |
| `pgxmock` | PostgreSQL mock | v2 |

```bash
go get github.com/stretchr/testify
go install github.com/vektra/mockery/v2@latest
```

---

## Test File Structure

Tests live alongside the code they test, in `_test.go` files.

```
internal/
├── service/
│   ├── user_service.go
│   └── user_service_test.go     # Unit tests
├── handler/
│   ├── user_handler.go
│   └── user_handler_test.go     # Integration tests (httptest)
└── repository/
    ├── user_repository.go
    └── user_repository_test.go  # Integration tests (test DB)
```

Integration tests that require a running database or external service use a **build tag**:

```go
//go:build integration

package repository_test
```

Run unit tests: `go test ./...`
Run all tests: `go test -tags=integration ./...`

---

## Coverage Requirement

**Minimum: 80% statement coverage** on all `internal/` packages.

CI enforces coverage with:

```bash
go test -race -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | grep total | awk '{print $3}'
```

Coverage thresholds are enforced per package. A single well-tested service package does not compensate for an untested one.

---

## Table-Driven Tests

Always use table-driven tests for functions with multiple input/output cases.

```go
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        {
            name:    "valid email",
            email:   "alice@example.com",
            wantErr: false,
        },
        {
            name:    "missing @ symbol",
            email:   "aliceexample.com",
            wantErr: true,
        },
        {
            name:    "empty string",
            email:   "",
            wantErr: true,
        },
        {
            name:    "unicode local part",
            email:   "аlice@example.com",
            wantErr: false,
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            err := ValidateEmail(tc.email)
            if tc.wantErr {
                require.Error(t, err)
            } else {
                require.NoError(t, err)
            }
        })
    }
}
```

---

## Unit Testing Services with Mocks

### Generating Mocks with `mockery`

Add a `//go:generate` directive in the file that defines the interface:

```go
//go:generate mockery --name=UserRepository --output=../mocks --outpkg=mocks
type UserRepository interface {
    Get(ctx context.Context, id uuid.UUID) (*User, error)
    Save(ctx context.Context, user *User) error
}
```

Run: `go generate ./...`

### Writing a Unit Test

```go
package service_test

import (
    "context"
    "testing"

    "github.com/google/uuid"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"

    "github.com/rbkl/my-service/internal/domain"
    "github.com/rbkl/my-service/internal/mocks"
    "github.com/rbkl/my-service/internal/service"
)

func TestUserService_Get(t *testing.T) {
    ctx := context.Background()
    id := uuid.New()

    t.Run("returns user when found", func(t *testing.T) {
        repo := mocks.NewUserRepository(t)
        repo.On("Get", ctx, id).Return(&domain.User{ID: id, Name: "Alice"}, nil)

        svc := service.NewUserService(repo)
        user, err := svc.Get(ctx, id)

        require.NoError(t, err)
        assert.Equal(t, "Alice", user.Name)
        repo.AssertExpectations(t)
    })

    t.Run("returns ErrNotFound when user does not exist", func(t *testing.T) {
        repo := mocks.NewUserRepository(t)
        repo.On("Get", ctx, id).Return(nil, domain.ErrNotFound)

        svc := service.NewUserService(repo)
        _, err := svc.Get(ctx, id)

        require.ErrorIs(t, err, domain.ErrNotFound)
    })
}
```

---

## HTTP Handler Testing

Test handlers using `net/http/httptest`. Never spin up a real server for handler tests.

```go
package handler_test

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/go-chi/chi/v5"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"

    "github.com/rbkl/my-service/internal/handler"
    "github.com/rbkl/my-service/internal/mocks"
)

func TestHandler_GetUser(t *testing.T) {
    t.Run("returns 200 with user", func(t *testing.T) {
        svc := mocks.NewUserService(t)
        svc.On("Get", mock.Anything, "user-123").Return(&domain.User{ID: "user-123", Name: "Alice"}, nil)

        h := handler.New(svc)
        r := chi.NewRouter()
        r.Get("/users/{userID}", h.GetUser)

        req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
        rec := httptest.NewRecorder()
        r.ServeHTTP(rec, req)

        assert.Equal(t, http.StatusOK, rec.Code)

        var body map[string]any
        require.NoError(t, json.Unmarshal(rec.Body.Bytes(), &body))
        assert.Equal(t, "Alice", body["name"])
    })

    t.Run("returns 404 when user not found", func(t *testing.T) {
        svc := mocks.NewUserService(t)
        svc.On("Get", mock.Anything, "missing").Return(nil, domain.ErrNotFound)

        h := handler.New(svc)
        r := chi.NewRouter()
        r.Get("/users/{userID}", h.GetUser)

        req := httptest.NewRequest(http.MethodGet, "/users/missing", nil)
        rec := httptest.NewRecorder()
        r.ServeHTTP(rec, req)

        assert.Equal(t, http.StatusNotFound, rec.Code)
    })
}
```

---

## Integration Testing with a Database

Integration tests run against a real PostgreSQL instance (typically via `docker-compose` in CI).

```go
//go:build integration

package repository_test

import (
    "context"
    "os"
    "testing"

    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/stretchr/testify/require"

    "github.com/rbkl/my-service/internal/database"
    "github.com/rbkl/my-service/internal/repository"
)

func setupDB(t *testing.T) *pgxpool.Pool {
    t.Helper()
    dsn := os.Getenv("TEST_DATABASE_URL")
    if dsn == "" {
        t.Skip("TEST_DATABASE_URL not set")
    }
    pool, err := database.NewPool(context.Background(), database.Config{DSN: dsn})
    require.NoError(t, err)
    t.Cleanup(func() { pool.Close() })
    return pool
}

func TestUserRepository_Get(t *testing.T) {
    pool := setupDB(t)
    repo := repository.NewUserRepository(pool)
    ctx := context.Background()

    // Seed test data
    user := seedUser(t, ctx, pool)

    t.Run("returns user by ID", func(t *testing.T) {
        got, err := repo.Get(ctx, user.ID)
        require.NoError(t, err)
        require.Equal(t, user.Email, got.Email)
    })
}
```

---

## Test Helpers and Fixtures

- Use `t.Helper()` in all test helper functions.
- Use `t.Cleanup()` for teardown — do not use `defer` in test setups.
- Use `testify/require` (fatal) for setup assertions; use `testify/assert` (non-fatal) for outcome assertions.
- Avoid global test state — each test must be independent and repeatable.

```go
func seedUser(t *testing.T, ctx context.Context, pool *pgxpool.Pool) *domain.User {
    t.Helper()
    user := &domain.User{
        ID:    uuid.New(),
        Email: fmt.Sprintf("test-%s@example.com", uuid.New().String()[:8]),
        Name:  "Test User",
    }
    _, err := pool.Exec(ctx,
        "INSERT INTO users (id, email, name) VALUES ($1, $2, $3)",
        user.ID, user.Email, user.Name,
    )
    require.NoError(t, err)
    t.Cleanup(func() {
        _, _ = pool.Exec(ctx, "DELETE FROM users WHERE id = $1", user.ID)
    })
    return user
}
```

---

## Prohibited Patterns

```go
// ❌ Skipping assertions — use require/assert
if err != nil {
    t.Error(err) // Non-fatal — test continues with potentially broken state
}

// ❌ Sleep-based timing — use channels or polling with timeout
time.Sleep(100 * time.Millisecond) // Flaky!

// ❌ Test-only build tags on production code
//go:build !test
func realImplementation() { ... }

// ❌ Shared mutable state between tests
var globalDB *pgxpool.Pool // Use t.Cleanup per test instead
```

---

_[← Error Handling](error-handling.md) · [API Design →](api-design.md)_
