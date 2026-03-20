# RBKL Development Standards

> **Audience:** Engineers and AI coding agents at RBKL. This wiki defines the official development standards, best practices, and required libraries for all software built within the organization.

## Table of Contents

### 1. [Overview & Philosophy](docs/overview.md)
- Mission and principles
- When to follow vs. when to deviate
- How this document is maintained

### 2. [Project Structure](docs/project-structure.md)
- Repository layout
- Monorepo vs. polyrepo strategy
- Naming conventions (files, packages, variables)
- Entry points and bootstrapping

### 3. [Web Servers & HTTP](docs/web-servers.md)
- Approved framework: `net/http` + `chi` router
- Middleware stack (CORS, request ID, timeout, recovery)
- Request/response lifecycle
- Graceful shutdown
- Health & readiness endpoints

### 4. [Logging](docs/logging.md)
- Approved library: `slog` (structured logging)
- Log levels and when to use them
- Correlation IDs and context propagation
- Sensitive data redaction
- Log output format (JSON in production, text in dev)

### 5. [Authentication & Authorization](docs/authentication.md)
- Identity provider: OIDC / Entra ID (Azure AD)
- JWT validation
- Role-based access control (RBAC)
- Service-to-service auth (mTLS / workload identity)
- Forbidden patterns

### 6. [Database Access](docs/database.md)
- Approved drivers: `pgx` (PostgreSQL), `database/sql` conventions
- ORM policy: `sqlc` for query generation
- Migrations: `golang-migrate`
- Connection pooling and timeouts
- Transaction handling

### 7. [Configuration Management](docs/configuration.md)
- Approved library: `viper`
- Environment variable conventions
- Secrets management (Azure Key Vault)
- Config validation at startup
- Feature flags

### 8. [Error Handling](docs/error-handling.md)
- Sentinel errors and `errors.Is` / `errors.As`
- Wrapping errors with context
- HTTP error responses (RFC 7807 Problem Details)
- Panic vs. error returns
- Error logging rules

### 9. [Testing](docs/testing.md)
- Standard library `testing` + `testify`
- Unit, integration, and end-to-end test structure
- Table-driven tests
- Test coverage requirements (≥ 80%)
- Mocking with `mockery`
- Contract testing

### 10. [API Design](docs/api-design.md)
- RESTful conventions
- Versioning strategy (`/v1/`, `/v2/`)
- OpenAPI 3.x specification requirements
- Pagination, filtering, sorting
- Idempotency

### 11. [Security](docs/security.md)
- Input validation and sanitization
- Dependency scanning (`govulncheck`, Dependabot)
- Secrets scanning (pre-commit hooks)
- OWASP top-10 mitigations
- Network policies and least privilege

### 12. [Dependency Management](docs/dependency-management.md)
- Go modules (`go.mod` / `go.sum`)
- Approved vs. restricted dependencies
- Vendoring policy
- Versioning and update cadence
- License compliance

### 13. [CI/CD Pipeline](docs/ci-cd.md)
- GitHub Actions as the only approved CI/CD platform
- Required pipeline stages
- Deployment environments (dev / staging / prod)
- Release process and semantic versioning
- Rollback procedures

---

## Quick Reference

| Concern | Approved Solution |
|---|---|
| Language | Go 1.22+ |
| HTTP Router | `chi` v5 |
| Structured Logging | `slog` (stdlib) |
| Configuration | `viper` |
| Database (SQL) | `pgx` v5 + `sqlc` |
| Migrations | `golang-migrate` |
| Identity | OIDC / Entra ID |
| JWT | `golang-jwt/jwt` v5 |
| Testing Assertions | `testify` |
| Mocking | `mockery` v2 |
| OpenAPI | `kin-openapi` / `oapi-codegen` |
| CI/CD | GitHub Actions |
| Secrets | Azure Key Vault |

---

_Last updated: 2026-03 · Maintained by the RBKL Platform Engineering team_