# Database Access

← [Back to README](../README.md)

## Approved Stack

| Component | Library | Version |
|---|---|---|
| PostgreSQL driver | `pgx` | v5 |
| SQL query generation | `sqlc` | latest |
| Migrations | `golang-migrate` | v4 |
| Connection pooling | `pgxpool` (bundled with pgx) | — |

> **Database of choice:** PostgreSQL 15+. No other relational database is approved without Platform Engineering sign-off.

---

## Connection Setup

### Using `pgxpool`

Always use `pgxpool` (not a raw `pgx.Conn`) for production services. The pool manages multiple connections, handles reconnection, and respects context cancellation.

```go
package database

import (
    "context"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

type Config struct {
    DSN             string
    MaxConns        int32
    MinConns        int32
    MaxConnLifetime time.Duration
    MaxConnIdleTime time.Duration
}

func NewPool(ctx context.Context, cfg Config) (*pgxpool.Pool, error) {
    poolCfg, err := pgxpool.ParseConfig(cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("parse database config: %w", err)
    }

    poolCfg.MaxConns = cfg.MaxConns           // Default: 10
    poolCfg.MinConns = cfg.MinConns           // Default: 2
    poolCfg.MaxConnLifetime = cfg.MaxConnLifetime // Default: 1h
    poolCfg.MaxConnIdleTime = cfg.MaxConnIdleTime // Default: 30m

    // Validate connectivity at startup
    pool, err := pgxpool.NewWithConfig(ctx, poolCfg)
    if err != nil {
        return nil, fmt.Errorf("create connection pool: %w", err)
    }

    if err := pool.Ping(ctx); err != nil {
        return nil, fmt.Errorf("ping database: %w", err)
    }

    return pool, nil
}
```

### Recommended Pool Sizing

| Service Type | MaxConns | MinConns |
|---|---|---|
| Low-traffic API | 5 | 1 |
| Standard API | 10 | 2 |
| High-traffic API | 25 | 5 |
| Background worker | 3 | 1 |

---

## Query Generation with `sqlc`

RBKL does **not** use ORM magic (no GORM, no `ent`). Instead, write raw SQL and use `sqlc` to generate type-safe Go code.

### Why `sqlc`?

- SQL is the source of truth — no surprises in generated queries.
- Full type safety — mismatches between Go types and SQL column types are caught at code-generation time.
- No runtime reflection overhead.
- Queries are readable, reviewable, and optimizable.

### `sqlc.yaml` Configuration

```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "internal/repository/queries/"
    schema: "migrations/"
    gen:
      go:
        package: "repository"
        out: "internal/repository"
        emit_json_tags: true
        emit_pointers_for_null_types: true
        emit_methods_with_db_argument: false
        emit_interface: true
```

### Query File Example (`internal/repository/queries/users.sql`)

```sql
-- name: GetUser :one
SELECT id, email, name, created_at, updated_at
FROM users
WHERE id = $1 AND deleted_at IS NULL;

-- name: ListUsers :many
SELECT id, email, name, created_at, updated_at
FROM users
WHERE deleted_at IS NULL
ORDER BY created_at DESC
LIMIT $1 OFFSET $2;

-- name: CreateUser :one
INSERT INTO users (id, email, name, created_at, updated_at)
VALUES ($1, $2, $3, NOW(), NOW())
RETURNING *;

-- name: SoftDeleteUser :exec
UPDATE users SET deleted_at = NOW() WHERE id = $1;
```

### Generated Repository Usage

```go
// sqlc generates this interface automatically
type Querier interface {
    GetUser(ctx context.Context, id uuid.UUID) (User, error)
    ListUsers(ctx context.Context, arg ListUsersParams) ([]User, error)
    CreateUser(ctx context.Context, arg CreateUserParams) (User, error)
    SoftDeleteUser(ctx context.Context, id uuid.UUID) error
}

// In your service layer:
func (s *UserService) Get(ctx context.Context, id uuid.UUID) (*domain.User, error) {
    row, err := s.db.GetUser(ctx, id)
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, domain.ErrNotFound
        }
        return nil, fmt.Errorf("get user %s: %w", id, err)
    }
    return toDomain(row), nil
}
```

---

## Migrations with `golang-migrate`

### File Naming

```
migrations/
├── 000001_create_users.up.sql
├── 000001_create_users.down.sql
├── 000002_add_users_email_index.up.sql
└── 000002_add_users_email_index.down.sql
```

Rules:
- Use sequential 6-digit zero-padded integers.
- Every `up` migration **must** have a matching `down` migration.
- Migrations are **immutable** once merged to `main`. Never edit an existing migration — create a new one.
- `down` migrations should fully reverse the `up` migration.

### Running Migrations

```go
package database

import (
    "fmt"

    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/pgx/v5"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func Migrate(databaseURL string) error {
    m, err := migrate.New("file://migrations", databaseURL)
    if err != nil {
        return fmt.Errorf("create migrator: %w", err)
    }
    defer m.Close()

    if err := m.Up(); err != nil && !errors.Is(err, migrate.ErrNoChange) {
        return fmt.Errorf("run migrations: %w", err)
    }

    return nil
}
```

Migrations run **automatically at application startup** in all environments except production. In production, migrations are a separate, gated pipeline step.

---

## Transaction Handling

### Standard Transaction Pattern

```go
func (r *UserRepository) TransferBalance(ctx context.Context, fromID, toID uuid.UUID, amount int64) error {
    return pgx.BeginTxFunc(ctx, r.pool, pgx.TxOptions{
        IsoLevel:   pgx.ReadCommitted,
        AccessMode: pgx.ReadWrite,
    }, func(tx pgx.Tx) error {
        qtx := r.queries.WithTx(tx)

        if err := qtx.DeductBalance(ctx, DeductBalanceParams{ID: fromID, Amount: amount}); err != nil {
            return fmt.Errorf("deduct from %s: %w", fromID, err)
        }
        if err := qtx.AddBalance(ctx, AddBalanceParams{ID: toID, Amount: amount}); err != nil {
            return fmt.Errorf("add to %s: %w", toID, err)
        }

        return nil
    })
}
```

- Use `pgx.BeginTxFunc` — it automatically commits on success or rolls back on error.
- Always specify the transaction isolation level explicitly.
- Keep transactions short — no network calls inside a transaction.

---

## Error Handling

Always translate database errors to domain errors at the repository boundary:

```go
import (
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgconn"
)

func toAppError(err error) error {
    if err == nil {
        return nil
    }
    if errors.Is(err, pgx.ErrNoRows) {
        return domain.ErrNotFound
    }
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        switch pgErr.Code {
        case "23505": // unique_violation
            return domain.ErrConflict
        case "23503": // foreign_key_violation
            return domain.ErrDependencyExists
        }
    }
    return err // Unexpected error — propagate as-is
}
```

---

## Prohibited Patterns

```go
// ❌ String interpolation in SQL queries — SQL injection risk
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email)

// ❌ Raw pgxpool.Conn — use the query interface
conn, _ := pool.Acquire(ctx)
conn.Exec(ctx, "SELECT ...")

// ❌ SELECT * — always list columns explicitly
db.Query("SELECT * FROM users")

// ❌ Ignoring row.Close() / rows.Close()
rows, _ := db.Query(ctx, sql)
// Missing: defer rows.Close()

// ❌ No context on database calls
db.QueryRow("SELECT ...")  // Use QueryRow(ctx, ...) instead
```

---

_[← Authentication](authentication.md) · [Configuration →](configuration.md)_
