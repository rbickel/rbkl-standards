# Project Structure

← [Back to README](../README.md)

## Repository Layout

Every RBKL Go service follows the same top-level layout. Consistency makes it trivial for engineers and coding agents to orient themselves in any repository.

```
my-service/
├── cmd/
│   └── server/
│       └── main.go          # Entry point — wire dependencies, start server
├── internal/
│   ├── config/              # Configuration loading & validation
│   ├── domain/              # Core business types (no external dependencies)
│   ├── handler/             # HTTP handlers (thin layer, no business logic)
│   ├── middleware/          # Custom HTTP middleware
│   ├── repository/          # Data access implementations
│   ├── service/             # Business logic orchestration
│   └── transport/           # Request/response DTOs and mapping
├── pkg/                     # Exported packages safe to use by other services
├── migrations/              # SQL migration files (golang-migrate format)
├── api/
│   └── openapi.yaml         # OpenAPI 3.x spec (source of truth)
├── docs/                    # Additional documentation (ADRs, runbooks)
├── scripts/                 # Build, seed, and utility scripts
├── .github/
│   └── workflows/           # CI/CD pipeline definitions
├── Dockerfile
├── docker-compose.yaml      # Local development environment
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

### Key Rules

- **`cmd/`** contains only `main.go` files. Each subdirectory is a separate binary. `main.go` wires everything together and starts the application — it contains no business logic.
- **`internal/`** is Go's way of preventing external packages from importing your internals. Always prefer `internal/` over `pkg/` unless you intentionally export the package for use by other services.
- **`pkg/`** is for genuinely reusable code. If in doubt, put it in `internal/`.
- **`domain/`** contains pure business types and interfaces. It **must not** import any external packages or framework code.
- **`repository/`** implements the storage interfaces defined in `domain/`. It imports `pgx`, Redis clients, etc.
- **`service/`** contains business logic. It depends on `domain/` interfaces only — never on concrete implementations.

---

## Monorepo vs. Polyrepo

RBKL uses a **polyrepo** strategy: one repository per deployable service or library.

**Rationale:**
- Independent CI/CD pipelines and release cadences.
- Smaller blast radius for dependency updates.
- Clearer ownership boundaries.

**Shared libraries** live in dedicated repositories under the `rbkl` GitHub organization (e.g., `rbkl/go-middleware`, `rbkl/go-auth`).

---

## Naming Conventions

### Files
| Type | Convention | Example |
|---|---|---|
| Go source | `snake_case.go` | `user_service.go` |
| Test file | `<file>_test.go` | `user_service_test.go` |
| Migration | `{version}_{description}.{up\|down}.sql` | `000001_create_users.up.sql` |
| Environment | `.env.{environment}` | `.env.local`, `.env.test` |

### Packages
- All lowercase, single word when possible: `handler`, `service`, `config`.
- Avoid stutter: `user.UserService` → `user.Service`.
- Avoid generic names: `util`, `common`, `helpers`, `misc`.

### Variables & Functions
- Use `camelCase` for unexported identifiers, `PascalCase` for exported.
- Acronyms are all-caps: `userID`, `HTTPClient`, `parseURL`.
- Receiver names are 1–2 letter abbreviations of the type: `(s *UserService)`, `(h *Handler)`.
- Boolean variables and functions use positive framing: `isActive`, `hasPermission` (not `notDisabled`).

### Constants and Errors
```go
// Constants: PascalCase if exported, camelCase if not
const MaxRetryAttempts = 3

// Sentinel errors: ErrXxx
var ErrNotFound = errors.New("not found")
var ErrUnauthorized = errors.New("unauthorized")
```

---

## Entry Point Pattern

`cmd/server/main.go` must follow this structure:

```go
package main

import (
    "context"
    "log/slog"
    "os"
    "os/signal"
    "syscall"

    "github.com/rbkl/my-service/internal/config"
    "github.com/rbkl/my-service/internal/server"
)

func main() {
    ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer cancel()

    cfg, err := config.Load()
    if err != nil {
        slog.Error("failed to load config", "error", err)
        os.Exit(1)
    }

    app, err := server.New(cfg)
    if err != nil {
        slog.Error("failed to initialize server", "error", err)
        os.Exit(1)
    }

    if err := app.Run(ctx); err != nil {
        slog.Error("server exited with error", "error", err)
        os.Exit(1)
    }
}
```

Key properties:
- Signal handling via `signal.NotifyContext` — not raw `os.Signal` channels.
- `os.Exit(1)` only in `main` — never in library code.
- All wiring happens in `server.New()`, not in `main`.

---

## Makefile

Every repository includes a `Makefile` with these standard targets:

```makefile
.PHONY: build test lint fmt generate migrate-up migrate-down docker-build

build:
	go build -o bin/server ./cmd/server

test:
	go test -race -coverprofile=coverage.out ./...

lint:
	golangci-lint run ./...

fmt:
	gofmt -w .
	goimports -w .

generate:
	go generate ./...

migrate-up:
	migrate -path migrations -database "$$DATABASE_URL" up

migrate-down:
	migrate -path migrations -database "$$DATABASE_URL" down 1

docker-build:
	docker build -t my-service:local .
```

---

_[← Overview](overview.md) · [Web Servers →](web-servers.md)_
