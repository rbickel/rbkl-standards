# Dependency Management

← [Back to README](../README.md)

## Package Manager

RBKL uses **`pnpm`** exclusively for all JavaScript/TypeScript projects. `npm` and `yarn` are not permitted.

**Why pnpm?**
- Efficient disk usage via content-addressable store (shared across projects).
- Strict mode by default — packages cannot access undeclared dependencies.
- First-class monorepo/workspace support.
- Faster installs than npm/yarn.

---

## Workspaces

The monorepo is managed with **pnpm workspaces** declared in `pnpm-workspace.yaml`:

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

Cross-workspace dependencies use the `workspace:` protocol:

```json
// apps/web/package.json
{
  "dependencies": {
    "@rbkl/ui": "workspace:*",
    "@rbkl/types": "workspace:*"
  }
}
```

---

## Node.js Version

Pin the Node.js version in `.nvmrc` and `package.json` engines field:

```
# .nvmrc
20.14.0
```

```json
// package.json (root)
{
  "engines": {
    "node": ">=20.14.0",
    "pnpm": ">=9.0.0"
  }
}
```

Use the LTS release of Node.js. Upgrade within 30 days of a new LTS version becoming active.

---

## Approved Core Dependencies

These packages are pre-approved for use in any RBKL application:

### Backend

| Category | Package | Version | Notes |
|---|---|---|---|
| HTTP framework | `fastify` | v4 | Required |
| Fastify CORS | `@fastify/cors` | latest | Required |
| Fastify Helmet | `@fastify/helmet` | latest | Required |
| Fastify Rate Limit | `@fastify/rate-limit` | latest | Required |
| Fastify Swagger | `@fastify/swagger` + `@fastify/swagger-ui` | latest | Required |
| Schema types | `@sinclair/typebox` | latest | With Fastify |
| Config validation | `zod` | v3 | Required |
| ORM | `@prisma/client` | v5 | Required |
| Logging | `pino` | v9 | Required |
| JWT / JWKS | `jose` | v5 | Required |
| Azure Identity | `@azure/identity` | v4 | Managed identity |
| Azure Key Vault | `@azure/keyvault-secrets` | v4 | Secrets |
| OpenTelemetry | `@opentelemetry/sdk-node` | latest | Tracing |

### Frontend

| Category | Package | Version | Notes |
|---|---|---|---|
| UI framework | `react` + `react-dom` | v18 | Required |
| Build tool | `vite` | v5 | Required |
| Router | `react-router-dom` | v6 | Required |
| State management | `zustand` | v4 | Preferred |
| Server state | `@tanstack/react-query` | v5 | Preferred |
| Forms | `react-hook-form` | v7 | Required |
| Zod integration | `@hookform/resolvers` | latest | With react-hook-form |
| MSAL (auth) | `@azure/msal-browser` + `@azure/msal-react` | v3 | Required |
| HTML sanitization | `dompurify` | v3 | When needed |
| HTTP sanitization | `@types/dompurify` | latest | Types |

---

## Restricted / Prohibited Packages

| Package | Reason |
|---|---|
| `lodash` | Use native ES2022+ methods; if needed use `lodash-es` for tree-shaking |
| `moment` | Use `date-fns` or `Temporal` API |
| `request` / `axios` | Use native `fetch` (Node 18+) or `undici` |
| `sequelize` / `typeorm` | Use Prisma |
| `winston` / `morgan` | Use Pino |
| `express` / `koa` | Use Fastify |
| `passport` | Use `jose` directly |
| `jsonwebtoken` (legacy) | Use `jose` v5 |
| `class-validator` | Use Zod |
| `next` (Next.js) | Not approved without Platform Engineering review |
| `redux` + `redux-toolkit` | Use Zustand; for complex state open a deviation request |

---

## Adding a New Dependency

Before adding any dependency:

1. **Check if native APIs cover it.** Node.js 20 includes `fetch`, `structuredClone`, `crypto`, and more.
2. **Check the approved list above.** Is there already an approved alternative?
3. **Evaluate the dependency:**
   - Weekly downloads (stability signal).
   - Last publish date (reject if > 12 months inactive).
   - Licence compatibility (see below).
   - Known vulnerabilities (`pnpm audit`).
   - Bundle size for frontend packages (use [bundlephobia.com](https://bundlephobia.com)).
4. **Pin the exact version** — no `^` or `~` ranges in `dependencies` (only in `devDependencies`).

```jsonc
// ✅ Good — pinned in dependencies
{ "dependencies": { "some-lib": "1.2.3" } }

// ❌ Bad — floating range
{ "dependencies": { "some-lib": "^1.2.3" } }
```

5. **Open a PR** — dependency additions are visible in `package.json` and reviewed.

---

## Version Update Cadence

| Dependency Type | Frequency |
|---|---|
| Security patches | Within 7 days of disclosure |
| Direct deps (patch/minor) | Monthly, via Dependabot PRs |
| Direct deps (major) | Quarterly, with explicit upgrade plan |
| Node.js runtime | Within 30 days of new LTS activation |
| `pnpm` | With each minor release |

---

## License Compliance

Only these licenses are permitted without explicit legal review:

| License | Permitted |
|---|---|
| MIT | ✅ Yes |
| Apache 2.0 | ✅ Yes |
| BSD 2-Clause | ✅ Yes |
| BSD 3-Clause | ✅ Yes |
| ISC | ✅ Yes |
| CC0 | ✅ Yes |
| MPL 2.0 | ⚠️ Review required |
| LGPL | ⚠️ Review required |
| GPL (any) | ❌ No |
| SSPL | ❌ No |
| Commons Clause | ❌ No |
| Proprietary | ❌ No (without legal review) |

Run license checks with:

```bash
pnpm add -D license-checker
npx license-checker --onlyAllow "MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;CC0-1.0"
```

---

_[← Security](security.md) · [CI/CD →](ci-cd.md)_
