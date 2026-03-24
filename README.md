# RBKL Development Standards

> **Audience:** Engineers and AI coding agents at RBKL. This wiki defines the official development standards, best practices, and required libraries for all software built within the organization.

## Must-Know Standards (read before coding)

- Stack + tooling: TypeScript in strict mode (no `any`/`@ts-ignore`), Node.js 20, React 18, Fastify v4, Pino v9, Prisma v5, pnpm workspaces only (no npm/yarn). Keep business logic in services and Prisma access inside repositories, not route handlers.
- Config: Validate all env vars with Zod at startup and exit on failure; use env vars as the single source; secrets come from Azure Key Vault/Managed Identity only; `.env.example` is required; never expose secrets with `VITE_`.
- AuthZ/AuthN: OIDC with Entra ID using `jose`/`@fastify/jwt`; validate issuer, audience, tenant, exp/nbf on every request; enforce RBAC roles per route; never bypass auth in dev; service-to-service auth uses Managed Identity; do not store tokens in `localStorage`.
- HTTP servers: Use Fastify plugins, TypeBox/JSON Schema validation at the boundary, explicit CORS allowlists (no `"*"`), and required `/healthz` + `/readyz` endpoints. Set reasonable timeouts/body limits and keep route handlers thin.
- API design: RESTful, versioned paths `/v1/...`, JSON only, errors as RFC 7807 problem+json, cursor pagination for lists, `DELETE` returns 204, support `Idempotency-Key` for retry-safe POSTs when needed.
- Logging: Pino only via `request.log`; redact auth headers/passwords/tokens/secrets; no `console.log` or string interpolation logs.
- Database: Prisma migrations for all schema changes, soft deletes via `deletedAt`, map Prisma errors to domain errors, transactions stay short (no external calls), and avoid raw SQL.
- Security: Apply `@fastify/helmet` + CSP and security headers, sanitize any HTML with DOMPurify before `dangerouslySetInnerHTML`, use rate limiting (`@fastify/rate-limit`), and never trust client-supplied URLs/headers for security decisions.
- Testing: Add/maintain unit, integration, and Playwright E2E coverage for new work; tests live with the code; minimum 80% coverage; never disable existing tests.
- CI/CD: GitHub Actions required with typecheck, lint, unit + integration (Postgres), build, audit, secrets scan, coverage gate, Docker build; Playwright runs on `main`; Conventional Commits drive releases.

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
