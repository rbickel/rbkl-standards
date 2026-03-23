# RBKL Development Standards

> **Audience:** Engineers and AI coding agents at RBKL. This wiki defines the official development standards, best practices, and required libraries for all software built within the organization.

## Table of Contents

### 1. [Overview & Philosophy](docs/overview.md)
- Mission and principles
- When to follow vs. when to deviate
- How this document is maintained

### 2. [Project Structure](docs/project-structure.md)
- Monorepo layout with `pnpm` workspaces
- `apps/` vs. `packages/` conventions
- Naming conventions (files, modules, variables)
- Entry points and bootstrapping

### 3. [Web Servers & HTTP](docs/web-servers.md)
- Approved framework: **Fastify** v4
- Plugin architecture and middleware stack
- Request/response lifecycle
- Graceful shutdown
- Health & readiness endpoints

### 4. [Logging](docs/logging.md)
- Approved library: **Pino** (structured logging)
- Log levels and when to use them
- Correlation IDs and async context propagation
- Sensitive data redaction
- Log output format (JSON in production, pino-pretty in dev)

### 5. [Authentication & Authorization](docs/authentication.md)
- Identity provider: OIDC / Entra ID (Azure AD)
- JWT validation with `jose`
- Role-based access control (RBAC)
- Service-to-service auth (managed identity)
- Forbidden patterns

### 6. [Database Access](docs/database.md)
- Approved ORM: **Prisma** + PostgreSQL
- Schema-first workflow
- Migrations with `prisma migrate`
- Connection pooling (PgBouncer / Prisma Accelerate)
- Transaction handling

### 7. [Configuration Management](docs/configuration.md)
- Environment variables with `dotenv`
- Schema validation with **Zod**
- Secrets management (Azure Key Vault)
- Config validation at startup
- Feature flags

### 8. [Error Handling](docs/error-handling.md)
- Custom error classes
- Fastify error hooks
- HTTP error responses (RFC 7807 Problem Details)
- Async/await error boundaries
- Error logging rules

### 9. [Testing](docs/testing.md)

Implement unit tests and integration tests everytime you implement a new feature. Consider existing tests as working and do never disable/comment them without explicit instructions. Follow testing standards from the company guidelines, and ensure all new code is covered by tests. If a new issue is found in existing code, fix it and add tests to cover that case

- **Vitest** for unit and integration tests
- **React Testing Library** for component tests
- **Supertest** for HTTP integration tests
- Test coverage requirements (≥ 80%)
- Mocking with `vi.mock`
- End-to-end testing with **Playwright**

### 10. [API Design](docs/api-design.md)
- RESTful conventions
- Versioning strategy (`/v1/`, `/v2/`)
- OpenAPI 3.x specification with `fastify-swagger`
- Pagination, filtering, sorting
- Idempotency

### 11. [Security](docs/security.md)
- Input validation and sanitization
- Dependency scanning (`npm audit`, Dependabot)
- Secrets scanning (pre-commit hooks)
- OWASP top-10 mitigations (including frontend XSS, CSRF)
- React-specific security (`DOMPurify`, CSP)

### 12. [Dependency Management](docs/dependency-management.md)
- `pnpm` workspaces
- Approved vs. restricted dependencies
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
| Backend Language | TypeScript (Node.js 20 LTS) |
| Frontend Language | TypeScript + React 18 |
| Package Manager | `pnpm` v9 |
| HTTP Framework | Fastify v4 |
| Frontend Build | Vite v5 |
| Structured Logging | Pino v9 |
| Configuration Validation | Zod v3 |
| Database ORM | Prisma v5 (PostgreSQL) |
| Identity | OIDC / Entra ID |
| JWT | `jose` v5 |
| Unit/Integration Testing | Vitest v1 |
| React Component Testing | React Testing Library |
| E2E Testing | Playwright |
| HTTP Integration Testing | Supertest |
| OpenAPI | `@fastify/swagger` + `@fastify/swagger-ui` |
| CI/CD | GitHub Actions |
| Secrets | Azure Key Vault |

---

_Last updated: 2026-03 · Maintained by the RBKL Platform Engineering team_
