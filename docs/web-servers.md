# Web Servers & HTTP

← [Back to README](../README.md)

## Approved Stack

| Component | Library | Version |
|---|---|---|
| HTTP framework | **Fastify** | v4 |
| OpenAPI / Swagger | `@fastify/swagger` + `@fastify/swagger-ui` | latest |
| CORS | `@fastify/cors` | latest |
| JWT plugin | `@fastify/jwt` (wraps `jose`) | latest |
| Static files | `@fastify/static` | latest |
| Request validation | Built-in (JSON Schema via `ajv`) | — |

> **Why Fastify?** Fastify is the fastest production-grade Node.js HTTP framework. It has a built-in schema-based request/response validator (no extra library needed), first-class TypeScript support, a structured plugin system, and Pino as its built-in logger.

---

## Server Setup

```typescript
// apps/api/src/server.ts
import Fastify, { FastifyInstance } from "fastify";
import cors from "@fastify/cors";
import { userRoutes } from "./routes/users.js";
import { authPlugin } from "./plugins/auth.js";
import { prismaPlugin } from "./plugins/prisma.js";
import type { Config } from "./config/index.js";

export async function buildServer(config: Config): Promise<FastifyInstance> {
  const server = Fastify({
    logger: {
      level: config.LOG_LEVEL,
      // pino-pretty in dev, raw JSON in production
      transport: config.NODE_ENV !== "production"
        ? { target: "pino-pretty", options: { colorize: true } }
        : undefined,
    },
    requestIdHeader: "x-request-id",
    requestIdLogLabel: "requestId",
    disableRequestLogging: false,
  });

  // ── Global plugins ────────────────────────────────────────────────────────
  await server.register(cors, {
    origin: config.ALLOWED_ORIGINS,
    methods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    allowedHeaders: ["Authorization", "Content-Type", "X-Request-ID"],
    credentials: true,
  });

  // ── Application plugins ───────────────────────────────────────────────────
  await server.register(prismaPlugin);  // Decorates server with server.prisma
  await server.register(authPlugin, { config }); // JWT validation

  // ── Routes ────────────────────────────────────────────────────────────────
  await server.register(userRoutes, { prefix: "/v1/users" });

  // ── Health endpoints ──────────────────────────────────────────────────────
  server.get("/healthz", { logLevel: "silent" }, async () => ({ status: "ok" }));
  server.get("/readyz", async (request, reply) => {
    try {
      await server.prisma.$queryRaw`SELECT 1`;
      return { status: "ready" };
    } catch (err) {
      request.log.warn({ err }, "Readiness check failed");
      return reply.status(503).send({ status: "unavailable" });
    }
  });

  return server;
}
```

---

## Plugin Architecture

Fastify uses a **plugin** system for all cross-cutting concerns. A plugin is a function that registers decorators, hooks, or routes on a Fastify scope.

### Plugin Pattern

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
    log: [{ emit: "event", level: "query" }],
  });

  await prisma.$connect();

  server.decorate("prisma", prisma);

  server.addHook("onClose", async () => {
    await prisma.$disconnect();
  });
});

export { prismaPlugin };
```

> **Always wrap plugins with `fastify-plugin`** (`fp`) when you want the decorators/hooks to be available in the parent scope. Without `fp`, Fastify encapsulates the plugin in its own scope.

---

## Request Validation with JSON Schema

Fastify validates requests using JSON Schema **before** the handler runs. Define schemas in separate files and attach them to routes.

```typescript
// apps/api/src/routes/users.ts
import { FastifyPluginAsync } from "fastify";
import { Type, Static } from "@sinclair/typebox"; // TypeBox for schema + TypeScript type
import { UserService } from "../services/user-service.js";

const CreateUserBody = Type.Object({
  email: Type.String({ format: "email", maxLength: 254 }),
  name: Type.String({ minLength: 2, maxLength: 100 }),
  role: Type.Union([
    Type.Literal("admin"),
    Type.Literal("editor"),
    Type.Literal("viewer"),
  ]),
});

type CreateUserBody = Static<typeof CreateUserBody>;

export const userRoutes: FastifyPluginAsync = async (server) => {
  const userService = new UserService(server.prisma, server.log);

  server.post<{ Body: CreateUserBody }>(
    "/",
    {
      schema: {
        body: CreateUserBody,
        response: {
          201: UserResponse,
          422: ProblemDetails,
        },
      },
      onRequest: [server.authenticate], // Auth hook from authPlugin
    },
    async (request, reply) => {
      const user = await userService.create(request.body);
      return reply.status(201).send(user);
    },
  );
};
```

> **`@sinclair/typebox`** is the approved library for defining JSON Schema that also generates TypeScript types. This avoids duplicating schemas and types.

---

## Hooks (Middleware)

Fastify uses **lifecycle hooks** instead of a middleware chain. Apply them at different scopes.

| Hook | Equivalent | Usage |
|---|---|---|
| `onRequest` | Pre-middleware | Authentication, rate limiting |
| `preValidation` | Before schema validation | Parse cookies, custom parsing |
| `preHandler` | Before handler | Authorization (RBAC), request enrichment |
| `onSend` | Before response sent | Modify response, add headers |
| `onError` | On error | Error formatting, logging |
| `onClose` | Server shutdown | Cleanup (DB disconnect) |

```typescript
// Global hook registered in buildServer
server.addHook("onRequest", async (request, reply) => {
  // Runs on every request
  request.log.info({ method: request.method, url: request.url }, "incoming request");
});

server.addHook("onError", async (request, reply, error) => {
  request.log.error({ err: error }, "request error");
});
```

---

## Graceful Shutdown

Fastify handles graceful shutdown via `server.close()`, which drains in-flight requests.

```typescript
// apps/api/src/index.ts
const shutdown = async (): Promise<void> => {
  server.log.info("Shutdown signal received — draining connections");
  try {
    await server.close(); // Waits for in-flight requests (up to closeGrace)
    process.exit(0);
  } catch (err) {
    server.log.error({ err }, "Error during shutdown");
    process.exit(1);
  }
};

process.on("SIGINT", shutdown);
process.on("SIGTERM", shutdown);
```

Configure the close grace period (default: none):

```typescript
const server = Fastify({
  closeGrace: 15_000, // 15 seconds to drain in-flight requests
});
```

---

## Health & Readiness Endpoints

Every service **must** expose these two endpoints (no authentication required):

### `GET /healthz` — Liveness

Returns `200 OK` if the Node.js process is alive.

```typescript
server.get("/healthz", { logLevel: "silent" }, async () => {
  return { status: "ok" };
});
```

### `GET /readyz` — Readiness

Returns `200 OK` only when all dependencies are ready.

```typescript
server.get("/readyz", async (request, reply) => {
  try {
    await server.prisma.$queryRaw`SELECT 1`;
    return { status: "ready" };
  } catch (err) {
    request.log.warn({ err }, "Readiness check failed");
    return reply.status(503).send({ status: "unavailable" });
  }
});
```

---

## Required Server Configuration

```typescript
const server = Fastify({
  // Timeouts
  connectionTimeout: 10_000,       // 10 seconds to establish connection
  keepAliveTimeout: 72_000,        // 72 seconds keep-alive (above ALB 60s)
  requestTimeout: 30_000,          // 30 seconds max request processing

  // Limits
  bodyLimit: 1_048_576,            // 1 MiB body limit
  maxParamLength: 100,             // URL param max length

  // Logging
  logger: pinoConfig,
  requestIdHeader: "x-request-id",
});
```

---

## CORS Policy

**Never use `origin: "*"` in production.** Always configure an explicit allowlist.

```typescript
await server.register(cors, {
  origin: (origin, callback) => {
    const allowed = config.ALLOWED_ORIGINS; // e.g. ["https://app.rbkl.com"]
    if (!origin || allowed.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error("Not allowed by CORS"), false);
    }
  },
  credentials: true,
  maxAge: 300,
});
```

---

_[← Project Structure](project-structure.md) · [Logging →](logging.md)_

