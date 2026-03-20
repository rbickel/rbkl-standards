# Dependency Management

← [Back to README](../README.md)

## Go Modules

All RBKL Go services use **Go modules** (`go.mod` / `go.sum`). The `go.sum` file is always committed and acts as a cryptographic verification of all dependencies.

```
my-service/
├── go.mod    # Module declaration, direct dependencies with versions
└── go.sum    # Checksums for all dependencies (direct + transitive)
```

---

## Module Naming

Module paths follow the GitHub repository path:

```go
module github.com/rbkl/my-service

go 1.22
```

---

## Approved vs. Restricted Dependencies

### Approved Core Dependencies

These libraries are pre-approved for use in any RBKL service:

| Category | Library | Version | Notes |
|---|---|---|---|
| HTTP Router | `github.com/go-chi/chi/v5` | v5.x | Preferred router |
| Config | `github.com/spf13/viper` | v1.x | Required |
| Validation | `github.com/go-playground/validator/v10` | v10.x | Input validation |
| Database | `github.com/jackc/pgx/v5` | v5.x | PostgreSQL only |
| Migrations | `github.com/golang-migrate/migrate/v4` | v4.x | Required with pgx |
| JWT | `github.com/golang-jwt/jwt/v5` | v5.x | Auth |
| JWKS | `github.com/MicahParks/keyfunc/v3` | v3.x | Auth |
| Testing | `github.com/stretchr/testify` | v1.x | Required |
| Mocks | `github.com/vektra/mockery/v2` | v2.x | Code generation |
| UUID | `github.com/google/uuid` | v1.x | IDs |
| OpenTelemetry | `go.opentelemetry.io/otel` | v1.x | Tracing |
| Azure Identity | `github.com/Azure/azure-sdk-for-go/sdk/azidentity` | v1.x | Managed identity |
| Azure Key Vault | `github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azsecrets` | v1.x | Secrets |

### Restricted/Prohibited Libraries

These libraries are **not approved** without explicit Platform Engineering review:

| Library | Reason |
|---|---|
| `github.com/jinzhu/gorm` / `gorm.io/gorm` | Use `sqlc` instead |
| `github.com/gomodule/redigo` | Use `github.com/redis/go-redis/v9` |
| `github.com/sirupsen/logrus` | Use stdlib `slog` |
| `go.uber.org/zap` | Use stdlib `slog` |
| `github.com/labstack/echo` | Use `chi` |
| `github.com/gin-gonic/gin` | Use `chi` |
| `github.com/dgrijalva/jwt-go` | Deprecated and unmaintained; use `golang-jwt/jwt/v5` |
| `math/rand` (for security) | Use `crypto/rand` for all security-sensitive random generation |

---

## Adding a New Dependency

Before adding any dependency:

1. **Check if the standard library covers it.** Go's stdlib is extensive — avoid unnecessary dependencies.
2. **Check if an approved library already covers it.** Avoid adding a second library that overlaps with an approved one.
3. **Evaluate the dependency:**
   - Last commit date (reject if > 12 months without activity and no maintainer statement).
   - Number of open critical issues.
   - License compatibility (see below).
   - Known vulnerabilities (run `govulncheck`).
4. **Pin the exact version** — no `@latest` or version ranges.

```bash
# ✅ Correct — pin exact version
go get github.com/some/library@v1.2.3

# ❌ Wrong — unpinned
go get github.com/some/library
```

5. **Open a PR** — dependency additions are visible in `go.mod` and reviewed as part of the PR.

---

## Vendoring Policy

RBKL does **not** vendor dependencies by default. The `go.sum` file provides integrity verification. Vendoring adds repository bloat and maintenance overhead.

**Exception:** Security-sensitive services or air-gapped environments may vendor dependencies. Document the reason in the repository README.

```bash
# Enable vendoring (only when explicitly required)
go mod vendor
```

---

## Version Update Cadence

| Dependency Type | Update Frequency |
|---|---|
| Security patches | Within 7 days of disclosure |
| Direct dependencies (minor/patch) | Monthly, via Dependabot PRs |
| Direct dependencies (major) | Quarterly, with explicit upgrade plan |
| Go toolchain | Within 1 month of a new minor release |

### Automated Updates

Dependabot handles routine updates. Configure it as follows:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "gomod"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "UTC"
    open-pull-requests-limit: 10
    groups:
      azure-sdk:
        patterns:
          - "github.com/Azure/*"
      otel:
        patterns:
          - "go.opentelemetry.io/*"
```

---

## License Compliance

RBKL services may only depend on libraries with the following licenses:

| License | Permitted |
|---|---|
| MIT | ✅ Yes |
| Apache 2.0 | ✅ Yes |
| BSD 2-Clause | ✅ Yes |
| BSD 3-Clause | ✅ Yes |
| ISC | ✅ Yes |
| MPL 2.0 | ⚠️ Review required |
| LGPL | ⚠️ Review required |
| GPL (any version) | ❌ No |
| SSPL | ❌ No |
| Commons Clause | ❌ No |
| Proprietary | ❌ No (without explicit legal review) |

Run `go-licenses` to generate a license report:

```bash
go install github.com/google/go-licenses@latest
go-licenses report ./... --template licenses.tpl
```

---

## Go Toolchain

Specify the minimum Go version in `go.mod`. Services should always target the **latest stable minor release** of Go.

```go
module github.com/rbkl/my-service

go 1.22

toolchain go1.22.5
```

The CI base image is updated within 30 days of each new Go minor release.

---

_[← Security](security.md) · [CI/CD →](ci-cd.md)_
