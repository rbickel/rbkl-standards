# Database Access

← [Back to README](../README.md)

## Approved Stack

| Component | Library | Version |
|---|---|---|
| ORM | **Prisma** | v5 |
| Database | PostgreSQL 15+ | — |
| Connection pooling | PgBouncer (external) or Prisma Accelerate | — |

> **Why Prisma?** Prisma provides a type-safe, schema-first ORM with an excellent migration system, auto-generated TypeScript types, and strong support for PostgreSQL. It eliminates raw SQL string hazards while keeping queries readable and explicit.

---

## Setup

```bash
pnpm add @prisma/client
pnpm add -D prisma
npx prisma init
```

---

## Prisma Schema

The `prisma/schema.prisma` file is the **single source of truth** for your database schema and generated types.

```prisma
// apps/api/prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String    @id @default(cuid())
  email     String    @unique
  name      String
  role      UserRole  @default(VIEWER)
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  deletedAt DateTime?

  @@map("users")
}

enum UserRole {
  ADMIN
  EDITOR
  VIEWER
}
```

### Schema Rules

- Every model has an `id` field using `@default(cuid())` or `@default(uuid())`.
- Soft deletes use a nullable `deletedAt` field — never hard-delete user data without review.
- Always define `createdAt` and `updatedAt` on mutable entities.
- Use `@@map("snake_case_table_name")` to keep table names `snake_case` even when the Prisma model is `PascalCase`.
- Define a `@@index` for every foreign key and commonly filtered column.

---

## Prisma Client Setup (Fastify Plugin)

```typescript
// apps/api/src/plugins/prisma.ts
import fp from "fastify-plugin";
import { FastifyPluginAsync } from "fastify";
import { PrismaClient } from "@prisma/client";

declare module "fastify" {
  interface FastifyInstance {
    prisma: PrismaClient;
  }
}

const prismaPlugin: FastifyPluginAsync = fp(async (server) => {
  const prisma = new PrismaClient({
    log:
      process.env.NODE_ENV === "development"
        ? [{ emit: "event", level: "query" }, "warn", "error"]
        : ["warn", "error"],
  });

  if (process.env.NODE_ENV === "development") {
    prisma.$on("query", (e) => {
      server.log.debug({ query: e.query, params: e.params, durationMs: e.duration }, "prisma query");
    });
  }

  await prisma.$connect();

  server.decorate("prisma", prisma);

  server.addHook("onClose", async () => {
    await prisma.$disconnect();
  });
});

export { prismaPlugin };
```

---

## Repository Pattern

Wrap Prisma calls in repository classes to keep routes and services free of ORM details, and to make testing easier.

```typescript
// apps/api/src/repositories/user-repository.ts
import { PrismaClient, User, Prisma } from "@prisma/client";
import { NotFoundError, ConflictError } from "../domain/errors.js";

export class UserRepository {
  constructor(private readonly db: PrismaClient) {}

  async findById(id: string): Promise<User> {
    const user = await this.db.user.findFirst({
      where: { id, deletedAt: null },
    });
    if (!user) throw new NotFoundError(`User ${id} not found`);
    return user;
  }

  async findAll(params: { limit: number; cursor?: string }): Promise<User[]> {
    return this.db.user.findMany({
      where: { deletedAt: null },
      take: params.limit,
      ...(params.cursor && {
        skip: 1,
        cursor: { id: params.cursor },
      }),
      orderBy: { createdAt: "desc" },
    });
  }

  async create(data: Prisma.UserCreateInput): Promise<User> {
    try {
      return await this.db.user.create({ data });
    } catch (err) {
      if (
        err instanceof Prisma.PrismaClientKnownRequestError &&
        err.code === "P2002"
      ) {
        throw new ConflictError("Email already exists");
      }
      throw err;
    }
  }

  async softDelete(id: string): Promise<void> {
    await this.db.user.update({
      where: { id },
      data: { deletedAt: new Date() },
    });
  }
}
```

### Prisma Error Codes to Domain Errors

Always translate Prisma errors to domain errors at the repository boundary:

| Prisma Code | Meaning | Domain Error |
|---|---|---|
| `P2002` | Unique constraint violation | `ConflictError` |
| `P2003` | Foreign key constraint violation | `DependencyError` |
| `P2025` | Record not found (update/delete) | `NotFoundError` |

---

## Migrations

Use `prisma migrate` for all schema changes.

### Development Workflow

```bash
# After editing schema.prisma:
npx prisma migrate dev --name create_users_table
# This creates a migration file AND applies it to the dev DB

# Generate/regenerate the Prisma Client after schema changes
npx prisma generate
```

### Production Deployment

```bash
# Apply pending migrations to production (no prompt, no dev-only features)
npx prisma migrate deploy
```

**Rules:**
- Never edit a migration file after it has been committed to `main`.
- Every schema change goes through a new migration.
- `prisma migrate deploy` runs automatically in the CI/CD pipeline before starting the application.
- Migrations are run in a separate step with its own approval gate in production (see [CI/CD](ci-cd.md)).

---

## Transactions

Use `prisma.$transaction()` for multi-step operations that must be atomic.

```typescript
// Sequential transaction (auto-rollback on error)
const [user, profile] = await this.db.$transaction([
  this.db.user.create({ data: userData }),
  this.db.profile.create({ data: profileData }),
]);

// Interactive transaction (for conditional logic)
await this.db.$transaction(async (tx) => {
  const from = await tx.account.update({
    where: { id: fromId },
    data: { balance: { decrement: amount } },
  });

  if (from.balance < 0) {
    throw new Error("Insufficient balance"); // Triggers rollback
  }

  await tx.account.update({
    where: { id: toId },
    data: { balance: { increment: amount } },
  });
});
```

- Keep transactions as short as possible — no external HTTP calls inside a transaction.
- Use interactive transactions (`$transaction(async (tx) => {...})`) when you need to inspect intermediate results.

---

## Prohibited Patterns

```typescript
// ❌ Raw SQL string interpolation — SQL injection risk
await prisma.$queryRawUnsafe(`SELECT * FROM users WHERE email = '${email}'`);

// ❌ Using Prisma client directly in route handlers
server.get("/users/:id", async (req) => {
  return server.prisma.user.findFirst({ where: { id: req.params.id } }); // Bypass repository
});

// ❌ Catching all errors generically
try {
  await repo.create(data);
} catch {
  throw new Error("Something went wrong"); // Loses error type
}

// ❌ SELECT * equivalent — always specify select/include explicitly for large models
await prisma.user.findMany(); // Returns all fields including sensitive ones
```

---

_[← Authentication](authentication.md) · [Configuration →](configuration.md)_
