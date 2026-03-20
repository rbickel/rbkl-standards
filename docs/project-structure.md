# Project Structure

← [Back to README](../README.md)

## Monorepo Layout

RBKL uses a **monorepo** strategy managed with **pnpm workspaces**. All applications and shared packages for a product live in a single repository. This enables atomic changes across the stack, shared tooling, and easy dependency sharing.

```
my-product/
├── apps/
│   ├── api/                     # Fastify backend service
│   │   ├── src/
│   │   │   ├── config/          # Config loading & Zod validation
│   │   │   ├── domain/          # Core business types & interfaces
│   │   │   ├── plugins/         # Fastify plugins (auth, db, etc.)
│   │   │   ├── routes/          # Route handlers (thin layer)
│   │   │   ├── services/        # Business logic
│   │   │   ├── repositories/    # Data access (Prisma wrappers)
│   │   │   ├── middleware/      # Custom Fastify hooks & decorators
│   │   │   └── index.ts         # Entry point
│   │   ├── prisma/
│   │   │   ├── schema.prisma    # Prisma schema (source of truth)
│   │   │   └── migrations/      # Generated migration files
│   │   ├── tests/
│   │   │   ├── unit/
│   │   │   └── integration/
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── web/                     # React frontend (Vite)
│       ├── src/
│       │   ├── components/      # Reusable UI components
│       │   ├── features/        # Feature-scoped modules (co-located)
│       │   ├── hooks/           # Shared React hooks
│       │   ├── lib/             # Utility functions & API client
│       │   ├── pages/           # Route-level page components
│       │   ├── stores/          # Zustand state stores
│       │   ├── types/           # Shared TypeScript types
│       │   └── main.tsx         # Entry point
│       ├── public/
│       ├── tests/
│       ├── index.html
│       ├── vite.config.ts
│       ├── package.json
│       └── tsconfig.json
│
├── packages/
│   ├── ui/                      # Shared React component library
│   ├── types/                   # Shared TypeScript types (API contracts)
│   └── utils/                   # Shared pure utility functions
│
├── .github/
│   └── workflows/               # CI/CD pipeline definitions
├── .eslintrc.cjs                # Root ESLint config (extended by each app)
├── .prettierrc                  # Root Prettier config
├── docker-compose.yaml          # Local development environment
├── pnpm-workspace.yaml
├── package.json                 # Root package (dev tooling only)
├── turbo.json                   # Turborepo build pipeline
└── README.md
```

### Key Rules

- **`apps/`** contains deployable applications. Each has its own `package.json`, `tsconfig.json`, and build output.
- **`packages/`** contains shared libraries consumed by apps. They are never deployed independently.
- **`packages/types`** is the single source of truth for API request/response types shared between `api` and `web`.
- Business logic lives in `services/` — route handlers must never contain business logic.
- `repositories/` are the only files that import from Prisma's generated client.

---

## Tooling

| Tool | Purpose |
|---|---|
| `pnpm` v9 | Package manager and workspace orchestration |
| `Turborepo` | Build pipeline caching and task dependency graph |
| `TypeScript` 5.x | Language (strict mode required) |
| `ESLint` + `eslint-config-rbkl` | Linting |
| `Prettier` | Code formatting |
| `Vite` v5 | Frontend build tool and dev server |

---

## TypeScript Configuration

### Root `tsconfig.base.json`

All apps extend this base:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": false,
    "esModuleInterop": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

**`strict: true` is non-negotiable.** It enables: `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, and more.

---

## Naming Conventions

### Files

| Type | Convention | Example |
|---|---|---|
| TypeScript source | `kebab-case.ts` | `user-service.ts` |
| React component | `PascalCase.tsx` | `UserCard.tsx` |
| Test file | `*.test.ts` or `*.spec.ts` | `user-service.test.ts` |
| Prisma migration | Auto-generated | `20260320_create_users/` |
| Environment | `.env.{environment}` | `.env.local`, `.env.test` |

### Variables, Functions, Classes

| Identifier | Convention | Example |
|---|---|---|
| Variables & functions | `camelCase` | `getUserById`, `isActive` |
| Classes & interfaces | `PascalCase` | `UserService`, `CreateUserDto` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_ATTEMPTS` |
| React components | `PascalCase` | `UserCard`, `AuthProvider` |
| Type/interface props | `camelCase` | `userId`, `createdAt` |
| Enum values | `PascalCase` | `UserRole.Admin` |
| Files (non-React) | `kebab-case` | `user-service.ts` |

### Interfaces vs. Types

- Use `interface` for object shapes that may be extended.
- Use `type` for unions, intersections, and computed types.
- Never use `I` prefix on interfaces (`IUserService` → `UserService`).

---

## Entry Point Pattern (Backend)

`apps/api/src/index.ts` must follow this structure:

```typescript
import { buildServer } from "./server.js";
import { loadConfig } from "./config/index.js";
import { logger } from "./lib/logger.js";

async function main(): Promise<void> {
  const config = loadConfig(); // Throws on invalid config

  const server = await buildServer(config);

  const shutdown = async (): Promise<void> => {
    logger.info("Shutdown signal received");
    await server.close();
    process.exit(0);
  };

  process.on("SIGINT", shutdown);
  process.on("SIGTERM", shutdown);

  await server.listen({ port: config.PORT, host: "0.0.0.0" });
}

main().catch((err) => {
  logger.error({ err }, "Fatal startup error");
  process.exit(1);
});
```

---

## `package.json` Scripts

Every app includes these standard scripts:

```json
{
  "scripts": {
    "build": "tsc --build",
    "start": "node dist/index.js",
    "dev": "tsx watch src/index.ts",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "lint": "eslint src --ext .ts,.tsx",
    "lint:fix": "eslint src --ext .ts,.tsx --fix",
    "typecheck": "tsc --noEmit",
    "db:migrate": "prisma migrate deploy",
    "db:generate": "prisma generate",
    "db:studio": "prisma studio"
  }
}
```

---

_[← Overview](overview.md) · [Web Servers →](web-servers.md)_

