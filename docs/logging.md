# Logging

← [Back to README](../README.md)

## Approved Library

RBKL uses **Pino** for all structured logging in Node.js services. Pino is Fastify's built-in logger and is the fastest structured logger in the Node.js ecosystem.

```bash
pnpm add pino pino-pretty
pnpm add -D @types/pino
```

> **Do not use** `console.log`, `winston`, or `morgan` in production services. Pino is the single approved logging solution.

---

## Logger Initialization

Create the logger once and export it. Pass it to services via dependency injection — do not import a global logger instance.

```typescript
// apps/api/src/lib/logger.ts
import pino, { Logger, LoggerOptions } from "pino";
import type { Config } from "../config/index.js";

export function createLogger(config: Config): Logger {
  const options: LoggerOptions = {
    level: config.LOG_LEVEL, // "debug" | "info" | "warn" | "error"
    base: {
      service: config.SERVICE_NAME,
      env: config.NODE_ENV,
    },
    // Redact sensitive fields from all log output
    redact: {
      paths: ["req.headers.authorization", "*.password", "*.token", "*.secret"],
      censor: "[REDACTED]",
    },
    serializers: {
      err: pino.stdSerializers.err,
      error: pino.stdSerializers.err,
    },
    timestamp: pino.stdTimeFunctions.isoTime,
  };

  if (config.NODE_ENV === "production") {
    return pino(options);
  }

  return pino({
    ...options,
    transport: {
      target: "pino-pretty",
      options: { colorize: true, translateTime: "SYS:standard" },
    },
  });
}

export type { Logger };
```

Fastify uses the logger automatically when configured:

```typescript
const server = Fastify({
  logger: createLogger(config),
  requestIdHeader: "x-request-id",
});
```

---

## Log Levels

| Level | When to Use | Example |
|---|---|---|
| `trace` | Extremely verbose — never in production | Function entry/exit |
| `debug` | Diagnostic info useful during development | Request body, cache hit/miss, SQL query |
| `info` | Normal operational events | Server started, request completed, user logged in |
| `warn` | Something unexpected but recoverable | Retry attempted, deprecated endpoint called |
| `error` | Operation failed and requires attention | DB connection failed, unhandled promise rejection |
| `fatal` | Process cannot continue | Startup failure, catastrophic state |

### Log Level per Environment

| Environment | Minimum Level |
|---|---|
| `local` | `debug` |
| `dev` | `debug` |
| `staging` | `info` |
| `production` | `info` |

**Never use `error` for 4xx HTTP responses.** A missing resource or invalid request is a client error — use `info` or `warn`.

---

## Structured Logging

Always use key-value objects in log calls, never string concatenation.

```typescript
// ✅ Good — structured, parseable
request.log.info({
  userId: user.id,
  action: "user.created",
  durationMs: Date.now() - start,
});

// ❌ Bad — unstructured string
logger.info(`User ${user.id} created in ${elapsed}ms`);
```

### Standard Field Names

| Field | Type | Description |
|---|---|---|
| `requestId` | string | Unique request ID (from `X-Request-ID`) |
| `traceId` | string | OpenTelemetry trace ID |
| `spanId` | string | OpenTelemetry span ID |
| `userId` | string | Authenticated user ID |
| `service` | string | Service name |
| `env` | string | Deployment environment |
| `err` | Error | Error object (use `err`, not `error`) |
| `durationMs` | number | Duration in milliseconds |
| `method` | string | HTTP method |
| `url` | string | Request URL (no sensitive query params) |
| `statusCode` | number | HTTP response status |

---

## Request Context

In Fastify, each request has its own child logger (`request.log`) automatically created with the request ID. Always use `request.log` inside route handlers and hooks.

For services deeper in the call stack, inject the logger via constructor:

```typescript
// ✅ Good — logger injected into service
class UserService {
  constructor(
    private readonly db: UserRepository,
    private readonly log: Logger,
  ) {}

  async create(data: CreateUserDto): Promise<User> {
    this.log.debug({ data }, "Creating user");
    // ...
  }
}

// In route handler:
const userService = new UserService(server.prisma, request.log);
```

---

## Sensitive Data Redaction

Configure Pino's `redact` option at logger creation. The following paths **must always** be redacted:

- `req.headers.authorization`
- `*.password`, `*.passwordHash`
- `*.token`, `*.accessToken`, `*.refreshToken`
- `*.secret`, `*.apiKey`, `*.privateKey`
- `*.creditCard`, `*.ssn`

```typescript
// ❌ Do not pass sensitive values to logger even if redact is configured
logger.info({ password: user.password });

// ✅ Log only non-sensitive identifiers
logger.info({ userId: user.id, email: user.email });
```

---

## Log Output Format

### Production (JSON)

```json
{
  "level": "info",
  "time": "2026-03-20T12:34:56.789Z",
  "service": "user-service",
  "env": "production",
  "requestId": "abc-123",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "userId": "usr_01HX",
  "msg": "request completed",
  "method": "GET",
  "url": "/v1/users/usr_01HX",
  "statusCode": 200,
  "durationMs": 42
}
```

### Development (pino-pretty)

```
[12:34:56.789] INFO (user-service): request completed
    requestId: "abc-123"
    method: "GET"
    url: "/v1/users/usr_01HX"
    statusCode: 200
    durationMs: 42
```

---

## Prohibited Patterns

```typescript
// ❌ Never use console.log/warn/error in service code
console.log("something happened");
console.error(err);

// ❌ Never log string interpolations
logger.info(`User ${id} failed to load`);

// ❌ Never swallow errors without logging
try {
  await doSomething();
} catch {
  // Silent catch
}

// ❌ Never use the root logger in request handlers — use request.log
import { logger } from "../lib/logger.js";
logger.info("handler called"); // Loses request context
```

---

_[← Web Servers](web-servers.md) · [Authentication →](authentication.md)_
